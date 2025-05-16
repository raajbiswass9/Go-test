func TestCitiJiraFeedbackMutation_Success(t *testing.T) {
	defer monkey.UnpatchAll()

	// Patch: IsValidUser
	monkey.Patch(common.IsValidUser, func(_ graphql.ResolveParams) bool {
		return true
	})

	// Patch: Sanitize
	monkey.Patch(common.Sanitize, func(input interface{}) (interface{}, error) {
		return input, nil
	})

	// Patch: IsJiraURLValid
	monkey.Patch(common.IsJiraURLValid, func(urlPtr *string) bool {
		if urlPtr != nil && *urlPtr == "" {
			*urlPtr = "https://valid.atlassian.net/jira/"
		}
		return true
	})

	// Patch: IsInternalUser
	monkey.Patch(common.IsInternalUser, func(_ graphql.ResolveParams) bool {
		return false
	})

	// Patch: GetUserName
	monkey.Patch(common.GetUserName, func(_ graphql.ResolveParams) string {
		return "test-reporter"
	})

	// Patch: IsProjectKeyValid
	monkey.Patch(common.IsProjectKeyValid, func(_ string) bool {
		return true
	})

	// Patch: CreateJiraIssue
	monkey.Patch(mutation.CreateJiraIssue, func(input models.GeneralFeedback, reporter string) *jira.Issue {
		return &jira.Issue{
			Fields: &jira.IssueFields{
				Summary:     input.Title,
				Description: input.Description,
			},
		}
	})

	// Patch: models.CreateJiraIssue
	monkey.Patch(models.CreateJiraIssue, func(_ string, _ *jira.Issue) (string, error) {
		return "JIRA-123", nil
	})

	// Patch: models.GetJiraURL
	monkey.Patch(models.GetJiraURL, func(baseURL string, jiraID string) string {
		return fmt.Sprintf("%s/browse/%s", baseURL, jiraID)
	})

	// Prepare input args
	input := map[string]interface{}{
		"title":          "Bug in app",
		"description":    "App crashes on login",
		"projectKey":     "PRJ",
		"issueType":      "Bug",
		"reporter":       "original-reporter",
		"componentName":  "LoginModule",
		"teamId":         "Team-42",
		"labels":         []interface{}{"crash", "login"},
	}

	args := map[string]interface{}{
		"input":   input,
		"jiraURL": "",
	}

	// Execute mutation
	result, err := mutation.CitiJiraFeedbackMutation.Resolve(graphql.ResolveParams{
		Args: args,
	})

	assert.NoError(t, err)

	feedbackResult, ok := result.(models.JiraFeedbackResult)
	assert.True(t, ok, "Result should be of type JiraFeedbackResult")
	assert.Equal(t, "JIRA-123", feedbackResult.JiraID)
	assert.Equal(t, "https://valid.atlassian.net/jira//browse/JIRA-123", feedbackResult.JiraURL)
	assert.Equal(t, "New Jira Ticket created for the feedback", feedbackResult.ActionTaken)
}
