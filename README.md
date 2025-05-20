func TestCitiJiraFeedbackMutation_FallbackCreation(t *testing.T) {
    defer monkey.UnpatchAll()

    // 1. Create a dummy config struct for mocking
    type configStruct struct {
        JiraUsername string
    }
    
    // 2. Mock ConfigManager using proper reflection
    var originalConfig = config.ConfigManager // Save original if needed for cleanup
    config.ConfigManager = &configStruct{
        JiraUsername: "fallback-user@company.com",
    }
    defer func() { config.ConfigManager = originalConfig }() // Cleanup after test

    // ALTERNATIVE if you must use monkey.PatchInstanceMethod:
    /*
    monkey.PatchInstanceMethod(
        reflect.TypeOf(config.ConfigManager), 
        "JiraUsername", 
        func(_ *config.Config) string {
            return "fallback-user@company.com"
        })
    */

    // 3. Mock other dependencies
    monkey.Patch(models.CreateJiraIssue, func(_ string, _ *jira.Issue) (string, error) {
        return "", errors.New("initial failure")
    })

    var fallbackIssue *jira.Issue
    monkey.Patch(models.CreateJiraIssue, func(_ string, issue *jira.Issue) (string, error) {
        fallbackIssue = issue
        return "FALLBACK-123", nil
    })

    // 4. Mock authentication/validation (essential!)
    monkey.Patch(common.GetUserName, func(p graphql.ResolveParams) string {
        return "test-user@company.com"
    })
    monkey.Patch(common.IsValidUser, func(p graphql.ResolveParams) bool {
        return true
    })
    monkey.Patch(common.IsJiraURLValid, func(_ *string) bool {
        return true
    })

    // 5. Execute test
    ctx := context.WithValue(context.Background(), models.ContextKeyUser, "test-user@company.com")
    args := map[string]interface{}{
        "input": map[string]interface{}{
            "title":       "Fallback Test",
            "description": "Test fallback creation",
            "projectKey":  "FALL",
            "issueType":   "Bug",
        },
        "jiraURL": "",
    }

    result, err := mutation.CitiJiraFeedbackMutation.Resolve(graphql.ResolveParams{
        Args:    args,
        Context: ctx,
    })

    // 6. Assertions
    require.NoError(t, err)
    assert.NotNil(t, fallbackIssue)
    assert.Equal(t, "fallback-user@company.com", fallbackIssue.Fields.Reporter.Name)
    assert.Equal(t, "FALLBACK-123", result.(models.JiraFeedbackResult).JiraID)
}
