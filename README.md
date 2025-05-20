func TestCitiJiraFeedbackMutation_FallbackCreation(t *testing.T) {
    defer monkey.UnpatchAll()

    // ===== REQUIRED MOCKS =====
    // 1. Authentication
    testUser := "test-user@company.com"
    monkey.Patch(common.GetUserName, func(p graphql.ResolveParams) string {
        if val := p.Context.Value(models.ContextKeyUser); val != nil {
            return val.(string)
        }
        return testUser
    })
    monkey.Patch(common.IsValidUser, func(_ graphql.ResolveParams) bool { return true })
    monkey.Patch(common.IsInternalUser, func(_ graphql.ResolveParams) bool { return false })

    // 2. Config
    monkey.PatchInstanceMethod(reflect.TypeOf(config.ConfigManager), "JiraUsername", 
        func(_ *config.Config) string { return "fallback-user@company.com" })

    // 3. Jira Operations
    monkey.Patch(models.CreateJiraIssue, func(_ string, _ *jira.Issue) (string, error) {
        return "", errors.New("initial failure") // First attempt fails
    })
    var fallbackIssue *jira.Issue
    monkey.Patch(models.CreateJiraIssue, func(_ string, issue *jira.Issue) (string, error) {
        fallbackIssue = issue // Capture for verification
        return "FALLBACK-123", nil // Fallback succeeds
    })

    // 4. Validations
    monkey.Patch(common.IsJiraURLValid, func(_ *string) bool { return true })
    monkey.Patch(common.IsProjectKeyValid, func(_ string) bool { return true })
    monkey.Patch(common.Sanitize, func(i interface{}) (interface{}, error) { return i, nil })

    // 5. Logger
    var logMessages []string
    monkey.Patch(logger.Log.Info, func(args ...interface{}) {
        logMessages = append(logMessages, fmt.Sprint(args...))
    })

    // ===== TEST EXECUTION =====
    ctx := context.WithValue(context.Background(), models.ContextKeyUser, testUser)
    args := map[string]interface{}{
        "input": map[string]interface{}{
            "title":       "Fallback Test",
            "description": "Test fallback creation",
            "projectKey":  "FALL",
            "issueType":   "Bug",
            "reporter":    "original-user",
        },
        "jiraURL": "https://test.atlassian.net",
    }

    result, err := mutation.CitiJiraFeedbackMutation.Resolve(graphql.ResolveParams{
        Args:    args,
        Context: ctx,
    })

    // ===== VERIFICATIONS =====
    assert.NoError(t, err)
    assert.NotNil(t, result)
    assert.Contains(t, logMessages, "Into the fallback")
    
    // Verify fallback issue properties
    require.NotNil(t, fallbackIssue)
    assert.Equal(t, "fallback-user@company.com", fallbackIssue.Fields.Reporter.Name)
    assert.Equal(t, "Test fallback creation", fallbackIssue.Fields.Description)
    assert.Equal(t, "FALL", fallbackIssue.Fields.Project.Key)

    // Verify final result
    feedbackResult := result.(models.JiraFeedbackResult)
    assert.Equal(t, "FALLBACK-123", feedbackResult.JiraID)
    assert.Contains(t, feedbackResult.JiraURL, "FALLBACK-123")
}
