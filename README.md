func TestCitiJiraFeedbackMutation_Success(t *testing.T) {
    // Patch all the dependencies
    monkey.Patch(common.IsValidUser, func(params graphql.ResolveParams) bool {
        return true
    })

    // FIXED: Changed signature to match original
    monkey.Patch(common.Sanitize, func(args map[string]interface{}) (map[string]interface{}, error) {
        return args, nil
    })

    monkey.Patch(common.IsJiraURLValid, func(url *string) bool {
        return true
    })

    monkey.Patch(common.IsInternalUser, func(params graphql.ResolveParams) bool {
        return false
    })

    monkey.Patch(common.GetUserName, func(params graphql.ResolveParams) string {
        return "test-reporter"
    })

    monkey.Patch(common.IsProjectKeyValid, func(projectKey string) bool {
        return true
    })

    monkey.Patch(mutation.CreateJiraIssue, func(input models.GeneralFeedback, reporter string) *jira.Issue {
        return &jira.Issue{}
    })

    monkey.Patch(models.CreateJiraIssue, func(url string, issue *jira.Issue) (string, error) {
        return "JIRA-123", nil
    })

    monkey.Patch(models.GetJiraURL, func(baseURL, key string) string {
        return baseURL + "/browse/" + key
    })

    defer monkey.UnpatchAll()

    input := map[string]interface{}{
        "input": map[string]interface{}{
            "projectKey":    "PROJ",
            "title":         "Bug found",
            "description":   "Something broke.",
            "labels":       []string{"bug"},
            "issueType":     "Bug",
            "componentName": "UI",
            "teamId":        "team-123",
        },
        "jiraURL": "https://myjira.com",
    }

    params := graphql.ResolveParams{
        Args: input,
    }

    res, err := mutation.CitiJiraFeedbackMutation.Resolve(params)
    assert.NoError(t, err)

    result, ok := res.(models.JiraFeedbackResult)
    assert.True(t, ok)
    assert.Equal(t, "JIRA-123", result.JiraID)
    assert.Equal(t, "https://myjira.com/browse/JIRA-123", result.JiraURL)
    assert.Equal(t, "New Jira Ticket created for the feedback", result.ActionTaken)
}
