# Designer repository modes (generated)

| Rule ID | Required condition | If false | Message |
| --- | --- | --- | --- |
| designer.repository.KnownMode | repositoryMode is manual, capture or capture_and_commit | Block before launch | Unknown repositoryMode. Use manual, capture or capture_and_commit. |
| designer.repository.RepositoryRequired | manual, or saved repository settings are configured and validated | Block before launch | Automatic capture/commit requires saved valid repository settings. No silent fallback to manual. |
| designer.repository.CommentScope | comment omitted, or repositoryMode=capture_and_commit | Block before launch | comment belongs only to capture_and_commit; capture does not create a repository version. |
