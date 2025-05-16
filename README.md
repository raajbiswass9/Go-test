func TestCitiJiraFeedbackMutation_Success(t *testing.T) {
	defer monkey.UnpatchAll()

	monkey.Patch(common.IsValidUser, func(_ graphql.ResolveParams) bool {
		return true
	})

	monkey.Patch(common.Sanitize, func(input interface{}) (interface{}, error) {
		return input, nil
	})

	monkey.Patch(common.IsJiraURLValid, func(jiraURL *string) bool {
		if jiraURL == nil {
			return false
		}
		if *jiraURL == "" {
			*jiraURL = "https://valid.atlassian.net/jira/"
		}
		return true
	})

	monkey.Patch(common.IsInternalUser, func(_ graphql.ResolveParams) bool {
		return false
	})

	monkey.Patch(common.GetUserName, func(_ graphql.ResolveParams) string {
		return "test-reporter"
	})

	monkey.Patch(common.IsProjectKeyValid, func(_ string) bool {
		return true
	})

	monkey.Patch(mutation.CreateJiraIssue, func(input models.GeneralFeedback, reporter string) *jira.Issue {
		return &jira.Issue{
			Fields: &jira.IssueFields{
				Summary:     input.Title,
				Description: input.Description,
			},
		}
	})

	monkey.Patch(models.CreateJiraIssue, func(_ string, _ *jira.Issue) (string, error) {
		return "JIRA-123", nil
	})

	monkey.Patch(models.GetJiraURL, func(baseURL string, jiraID string) string {
		return baseURL + "browse/" + jiraID
	})

	args := map[string]interface{}{
		"input": map[string]interface{}{
			"title":         "Bug in app",
			"description":   "App crashes on login",
			"projectKey":    "PRJ",
			"issueType":     "Bug",
			"reporter":      "original-reporter",
			"componentName": "LoginModule",
			"teamId":        "Team-42",
			"labels":        []interface{}{"crash", "login"},
		},
		"jiraURL": "https://valid.atlassian.net/jira/", // required to prevent nil pointer
	}

	result, err := mutation.CitiJiraFeedbackMutation.Resolve(graphql.ResolveParams{
		Args: args,
	})

	assert.NoError(t, err)

	expected := models.JiraFeedbackResult{
		JiraID:      "JIRA-123",
		JiraURL:     "https://valid.atlassian.net/jira/browse/JIRA-123",
		ActionTaken: "New Jira Ticket created for the feedback",
	}

	actual, ok := result.(models.JiraFeedbackResult)
	assert.True(t, ok)
	assert.Equal(t, expected, actual)
}
