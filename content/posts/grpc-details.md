---
author: "Or Elimelech"
date: 2022-05-08
title: Add Protobuf messages into gRPC errors.
tags: ['bigquery', 'data', 'google-cloud']
---

I had encountred a number of times for the need to give a structured message through gRPC errors
The default `Go` err or `status.Errorf` functions are simple strings by default.

```go
func (s *Server) LintFile(ctx context.Context, req *LintRequest) (*LintResponse, error) {
	lintRes, err := lint(ctx, req.GetFile())
	if err != nil {
		return nil, status.New(codes.FailedPrecondition, "File isn't valid").WithDetails(lintRes)
	}
	return &LintResponse{}, nil
}
```

The example above illustrates a situation where you need to know what exactly was wrong with the file,
The line number of where the error happened, what's the reason etc.
