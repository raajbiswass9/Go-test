func TestCitiJiraFeedbackMutation_Success(t *testing.T) {
    monkey.Patch(common.IsValidUser, func(params graphql.ResolveParams) bool {
        return true
    })

    monkey.Patch(common.Sanitize, func(rawInput interface{}) (interface{}, error) {
        return rawInput, nil
    })

    monkey.Patch(common.IsJiraURLValid, func(jiraURL *string) bool {
        if jiraURL != nil && *jiraURL == "" {
            *jiraURL = "https://example.atlassian.net/"
        }
        return true
    })

    monkey.Patch(common.IsInternalUser, func(params graphql.ResolveParams) bool {
        return false
    })

    monkey.Patch(common.IsProjectKeyValid, func(projectKey string) bool {
        return true
    })

    monkey.Patch(models.CreateJiraIssue, func(jiraURL string, issue *jira.Issue) (string, error) {
        return "JIRA-999", nil
    })

    monkey.Patch(models.GetJiraURL, func(baseURL string, jiraID string) string {
        return baseURL + "browse/" + jiraID
    })

    defer monkey.UnpatchAll()

    args := map[string]interface{}{
        "jiraURL": "https://example.atlassian.net/",
        "input": map[string]interface{}{
            "issueType":     "Bug",
            "description":   "Test description",
            "title":         "Test title",
            "projectKey":    "TEST",
            "labels":        []interface{}{"test-label"},
            "reporter":      "john.doe",
            "teamID":        "team-123",
            "componentName": "UI",
        },
    }

    params := graphql.ResolveParams{
        Args: args,
    }

    result, err := CitiJiraFeedbackMutation.Resolve(params)
    assert.NoError(t, err)

    output, ok := result.(models.JiraFeedbackResult)
    assert.True(t, ok, "Result should be JiraFeedbackResult")
    assert.Equal(t, "JIRA-999", output.JiraID)
    assert.Equal(t, "https://example.atlassian.net/browse/JIRA-999", output.JiraURL)
    assert.Equal(t, "New Jira Ticket created for the feedback", output.ActionTaken)
}
