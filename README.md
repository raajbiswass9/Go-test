package mutation_test

import (
	"fmt"
	"runtime/debug"
	"testing"

	"github.com/bouk/monkey"
	"github.com/stretchr/testify/assert"
	"github.com/your_project_path/common"
	"github.com/your_project_path/config"
	"github.com/your_project_path/models"
	"github.com/your_project_path/mutation"
	"github.com/your_project_path/schema"
	"github.com/graphql-go/graphql"
	"github.com/andygrunwald/go-jira"
)

func TestCitiJiraFeedbackMutation_Success(t *testing.T) {
	defer monkey.UnpatchAll()

	// Add recovery to identify panic cause
	defer func() {
		if r := recover(); r != nil {
			t.Fatalf("Test panicked: %v\n\nSTACK TRACE:\n%s", r, string(debug.Stack()))
		}
	}()

	// Patch all dependencies
	monkey.Patch(common.IsValidUser, func(_ graphql.ResolveParams) bool {
		return true
	})

	monkey.Patch(common.Sanitize, func(input interface{}) (interface{}, error) {
		return input, nil
	})

	monkey.Patch(common.IsJiraURLValid, func(urlPtr *string) bool {
		if urlPtr != nil && *urlPtr == "" {
			*urlPtr = "https://valid.atlassian.net/jira/"
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

	monkey.Patch(models.CreateJiraIssue, func(_ string, _ *jira.Issue) (string, error) {
		return "JIRA-123", nil
	})

	monkey.Patch(models.GetJiraURL, func(baseURL string, jiraID string) string {
		return fmt.Sprintf("%s/browse/%s", baseURL, jiraID)
	})

	// Input data
	input := map[string]interface{}{
		"title":         "Bug in app",
		"description":   "App crashes on login",
		"projectKey":    "PRJ",
		"issueType":     "Bug",
		"reporter":      "original-reporter",
		"componentName": "LoginModule",
		"teamID":        "Team-42", // NOTE: Was previously "teamId", fixed to "teamID"
		"labels":        []interface{}{"crash", "login"},
	}

	args := map[string]interface{}{
		"input":   input,
		"jiraURL": "",
	}

	// Run the mutation
	result, err := mutation.CitiJiraFeedbackMutation.Resolve(graphql.ResolveParams{
		Args: args,
	})

	// Assertions
	assert.NoError(t, err)
	feedbackResult, ok := result.(models.JiraFeedbackResult)
	assert.True(t, ok, "Result should be of type JiraFeedbackResult")
	assert.Equal(t, "JIRA-123", feedbackResult.JiraID)
	assert.Equal(t, "https://valid.atlassian.net/jira/browse/JIRA-123", feedbackResult.JiraURL)
	assert.Equal(t, "New Jira Ticket created for the feedback", feedbackResult.ActionTaken)
}
