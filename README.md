comments, _ := lo.TryOr(func() ([]Comment, error) {
    result := make([]Comment, 0)
    if issue.Fields.Comments != nil {
        for _, c := range issue.Fields.Comments.Comments {
            result = append(result, Comment{
                Author:    c.Author.DisplayName,
                Body:      c.Body,
                CreatedAt: c.Created,
            })
        }
    }
    return result, nil
}, []Comment{})
