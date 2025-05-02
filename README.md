package query

import (
	"testing"
	"github.com/graphql-go/graphql"
	"github.com/stretchr/testify/assert"
	"bou.ke/monkey"
	"feedback/models"
	"time"
)

func TestJiraIssuesQuery_WithIssueID(t *testing.T) {
	// Mock the validateAndSanitizeParams function
	monkey.Patch(validateAndSanitizeParams, func(params graphql.ResolveParams) (string, error) {
		return "https://test.jira.com", nil
	})

	// Mock the extractParams function
	monkey.Patch(extractParams, func(args map[string]interface{}) (string, string, int, int, error) {
		return "", "ASC", 1, 10, nil
	})

	// Mock the decodeInputCategories function
	monkey.Patch(decodeInputCategories, func(params graphql.ResolveParams, inputCategories []string) (map[string][]string, error) {
		return map[string][]string{
			"issueID": {"TEST-123", "TEST-456"},
		}, nil
	})

	// Mock the models.FindJiraIssues function with proper JiraID type
	monkey.Patch(models.FindJiraIssues, func(jiraURL string, inputFilters map[string][]string, sortOrder, searchText string, pageNumber, pageSize int) ([]models.JiraIssue, error) {
		assert.Equal(t, []string{"TEST-123", "TEST-456"}, inputFilters["issueID"])
		return []models.JiraIssue{
			{JiraID: models.JiraID("TEST-123")}, // Properly type-converted
			{JiraID: models.JiraID("TEST-456")}, // Properly type-converted
		}, nil
	})

	// Clean up all monkey patches after test
	defer monkey.UnpatchAll()

	// Create test params with issueID
	params := graphql.ResolveParams{
		Args: map[string]interface{}{
			"issueID":    []interface{}{"TEST-123", "TEST-456"},
			"sortOrder":  "ASC",
			"pageNumber": 1,
			"pageSize":   10,
			"jiraURL":    "https://test.jira.com",
		},
	}

	// Execute the query
	result, err := JiraIssuesQuery.Resolve(params)

	// Verify results
	assert.Nil(t, err)
	assert.NotNil(t, result)
	issues, ok := result.([]models.JiraIssue)
	assert.True(t, ok)
	assert.Len(t, issues, 2)
	assert.Equal(t, models.JiraID("TEST-123"), issues[0].JiraID) // Type-converted comparison
	assert.Equal(t, models.JiraID("TEST-456"), issues[1].JiraID) // Type-converted comparison
}
