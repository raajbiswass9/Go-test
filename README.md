package yourpackage

import (
	"errors"
	"testing"

	"github.com/bouk/monkey"
	"github.com/graphql-go/graphql"
	"github.com/stretchr/testify/assert"
	"yourmodule/models"
)

func TestJiraIssuesQuery_Success(t *testing.T) {
	// Patch validateAndSanitizeParams
	monkey.Patch(validateAndSanitizeParams, func(params graphql.ResolveParams) (string, error) {
		return "mocked-jira-url", nil
	})
	defer monkey.Unpatch(validateAndSanitizeParams)

	// Patch extractParams
	monkey.Patch(extractParams, func(args interface{}) (string, string, int, int, error) {
		return "searchText", "asc", 1, 10, nil
	})
	defer monkey.Unpatch(extractParams)

	// Patch decodeInputCategories
	monkey.Patch(decodeInputCategories, func(params graphql.ResolveParams, categories []string) (map[string][]string, error) {
		return map[string][]string{"projectKeyIn": {"PROJ1"}}, nil
	})
	defer monkey.Unpatch(decodeInputCategories)

	// Patch models.FindJiraIssues
	monkey.Patch(models.FindJiraIssues, func(jiraURL string, filters map[string][]string, sortOrder, searchText string, pageNumber, pageSize int) (interface{}, error) {
		return []string{"ISSUE-1", "ISSUE-2"}, nil
	})
	defer monkey.Unpatch(models.FindJiraIssues)

	params := graphql.ResolveParams{
		Args: map[string]interface{}{
			"projectKeyIn": []string{"PROJ1"},
		},
	}

	result, err := JiraIssuesQuery.Resolve(params)
	assert.NoError(t, err)
	assert.Equal(t, []string{"ISSUE-1", "ISSUE-2"}, result)
}

func TestJiraIssuesQuery_ValidateFails(t *testing.T) {
	monkey.Patch(validateAndSanitizeParams, func(params graphql.ResolveParams) (string, error) {
		return "", errors.New("user not valid")
	})
	defer monkey.Unpatch(validateAndSanitizeParams)

	params := graphql.ResolveParams{}
	result, err := JiraIssuesQuery.Resolve(params)
	assert.Error(t, err)
	assert.Nil(t, result)
}

func TestJiraIssuesQuery_DecodeInputCategoriesFails(t *testing.T) {
	monkey.Patch(validateAndSanitizeParams, func(params graphql.ResolveParams) (string, error) {
		return "mocked-url", nil
	})
	defer monkey.Unpatch(validateAndSanitizeParams)

	monkey.Patch(extractParams, func(args interface{}) (string, string, int, int, error) {
		return "searchText", "asc", 1, 10, nil
	})
	defer monkey.Unpatch(extractParams)

	monkey.Patch(decodeInputCategories, func(params graphql.ResolveParams, categories []string) (map[string][]string, error) {
		return nil, errors.New("decode error")
	})
	defer monkey.Unpatch(decodeInputCategories)

	params := graphql.ResolveParams{}
	result, err := JiraIssuesQuery.Resolve(params)
	assert.Error(t, err)
	assert.Nil(t, result)
}





..........

func TestJiraIssuesQuery_WithIssueID(t *testing.T) {
	// Patch validateAndSanitizeParams
	monkey.Patch(validateAndSanitizeParams, func(params graphql.ResolveParams) (string, error) {
		return "mocked-jira-url", nil
	})
	defer monkey.Unpatch(validateAndSanitizeParams)

	// Patch extractParams
	monkey.Patch(extractParams, func(args interface{}) (string, string, int, int, error) {
		return "", "", 1, 10, nil
	})
	defer monkey.Unpatch(extractParams)

	// Patch decodeInputCategories
	monkey.Patch(decodeInputCategories, func(params graphql.ResolveParams, categories []string) (map[string][]string, error) {
		// Validate issueID presence
		assert.Contains(t, params.Args, "issueID")
		assert.Equal(t, []string{"ISSUE-123"}, params.Args["issueID"])

		return map[string][]string{
			"issueID": {"ISSUE-123"},
		}, nil
	})
	defer monkey.Unpatch(decodeInputCategories)

	// Patch FindJiraIssues
	monkey.Patch(models.FindJiraIssues, func(jiraURL string, filters map[string][]string, sortOrder, searchText string, pageNumber, pageSize int) (interface{}, error) {
		assert.Equal(t, "mocked-jira-url", jiraURL)
		assert.Contains(t, filters, "issueID")
		assert.Equal(t, []string{"ISSUE-123"}, filters["issueID"])
		return []string{"ISSUE-123"}, nil
	})
	defer monkey.Unpatch(models.FindJiraIssues)

	params := graphql.ResolveParams{
		Args: map[string]interface{}{
			"issueID": []string{"ISSUE-123"},
		},
	}

	result, err := JiraIssuesQuery.Resolve(params)
	assert.NoError(t, err)
	assert.Equal(t, []string{"ISSUE-123"}, result)
}







