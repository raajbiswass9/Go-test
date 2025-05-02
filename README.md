package query

import (
	"testing"
	"time"
	"github.com/graphql-go/graphql"
	"github.com/stretchr/testify/assert"
	"bou.ke/monkey"
	"feedback/models"
)

func TestJiraIssuesQuery_WithIssueID(t *testing.T) {
	// Mock dependencies
	monkey.Patch(validateAndSanitizeParams, func(params graphql.ResolveParams) (string, error) {
		return "https://test.jira.com", nil
	})

	monkey.Patch(extractParams, func(args map[string]interface{}) (string, string, int, int, error) {
		return "", "ASC", 1, 10, nil
	})

	monkey.Patch(decodeInputCategories, func(params graphql.ResolveParams, inputCategories []string) (map[string][]string, error) {
		return map[string][]string{
			"issueID": {"TEST-123", "TEST-456"},
		}, nil
	})

	// Create properly initialized JiraIssue instances
	now := time.Now()
	mockIssue1 := models.JiraIssue{
		JiraID:        models.JiraID("TEST-123"), // Assuming JiraID is a string alias
		IssueType:     models.IssueType{Name: "Story"},
		Priority:      models.Priority{Name: "High"},
		Summary:       "Test Issue 1",
		Description:   "Test Description 1",
		CreatedTime:   now,
		UpdatedTime:   now,
		Status:        models.Status{Name: "Open"},
		// Initialize other required fields with minimal values
	}

	mockIssue2 := models.JiraIssue{
		JiraID:        models.JiraID("TEST-456"),
		IssueType:     models.IssueType{Name: "Bug"},
		Priority:      models.Priority{Name: "Medium"},
		Summary:       "Test Issue 2",
		Description:   "Test Description 2",
		CreatedTime:   now,
		UpdatedTime:   now,
		Status:        models.Status{Name: "In Progress"},
		// Initialize other required fields with minimal values
	}

	monkey.Patch(models.FindJiraIssues, func(jiraURL string, inputFilters map[string][]string, sortOrder, searchText string, pageNumber, pageSize int) ([]models.JiraIssue, error) {
		assert.Equal(t, []string{"TEST-123", "TEST-456"}, inputFilters["issueID"])
		return []models.JiraIssue{mockIssue1, mockIssue2}, nil
	})

	defer monkey.UnpatchAll()

	// Test execution
	params := graphql.ResolveParams{
		Args: map[string]interface{}{
			"issueID":    []interface{}{"TEST-123", "TEST-456"},
			"sortOrder":  "ASC",
			"pageNumber": 1,
			"pageSize":   10,
			"jiraURL":    "https://test.jira.com",
		},
	}

	result, err := JiraIssuesQuery.Resolve(params)
	assert.Nil(t, err)
	
	issues, ok := result.([]models.JiraIssue)
	assert.True(t, ok)
	assert.Len(t, issues, 2)
	assert.Equal(t, models.JiraID("TEST-123"), issues[0].JiraID)
	assert.Equal(t, models.JiraID("TEST-456"), issues[1].JiraID)
}
