func TestCitiJiraFeedbackMutation_Success(t *testing.T) {
	defer monkey.UnpatchAll()

	// Patch all dependencies
	monkey.Patch(common.IsValidUser, func(_ graphql.ResolveParams) bool { return true })
	monkey.Patch(common.Sanitize, func(input interface{}) (interface{}, error) { return input, nil })
	monkey.Patch(common.IsInternalUser, func(_ graphql.ResolveParams) bool { return false })
	monkey.Patch(common.GetUserName, func(_ graphql.ResolveParams) string { return "test-reporter" })
	monkey.Patch(common.IsProjectKeyValid, func(_ string) bool { return true })

	monkey.Patch(mutation.CreateJiraIssue, func(input models.GeneralFeedback, reporter string) *jira.Issue {
		return &jira.Issue{
			Key: "JIRA-123",
		}
	})
	monkey.Patch(models.CreateJiraIssue, func(_ string, _ *jira.Issue) (string, error) {
		return "JIRA-123", nil
	})
	monkey.Patch(models.GetJiraURL, func(baseURL string, jiraID string) string {
		return baseURL + "browse/" + jiraID
	})

	// 🔥 This is the important part
	monkey.Patch(common.IsJiraURLValid, func(jiraURL *string) bool {
		if jiraURL == nil {
			t.Fatal("jiraURL is nil") // crash guard
		}
		if *jiraURL == "" {
			*jiraURL = "https://example.atlassian.net/jira/"
		}
		return true
	})

	// Test input args
	args := map[string]interface{}{
		"input": map[string]interface{}{
			"title":         "Bug in login",
			"description":   "Crash on login attempt",
			"projectKey":    "PRJ",
			"issueType":     "Bug",
			"reporter":      "original-reporter",
			"componentName": "Login",
			"teamId":        "Team42",
			"labels":        []interface{}{"bug", "crash"},
		},
		"jiraURL": "https://example.atlassian.net/jira/", // prevent nil or empty
	}

	resParams := graphql.ResolveParams{
		Args: args,
	}

	// 🔍 Debug: ensure correct jiraURL value
	fmt.Printf("Test input jiraURL: %#v\n", args["jiraURL"])

	// Run mutation
	result, err := mutation.CitiJiraFeedbackMutation.Resolve(resParams)
	assert.NoError(t, err)

	// Assert
	expected := models.JiraFeedbackResult{
		JiraID:      "JIRA-123",
		JiraURL:     "https://example.atlassian.net/jira/browse/JIRA-123",
		ActionTaken: "New Jira Ticket created for the feedback",
	}
	actual, ok := result.(models.JiraFeedbackResult)
	assert.True(t, ok)
	assert.Equal(t, expected, actual)
}
