# Tekton Run Test

**This project is in the design stage. We are doing prototyping.**

Generate a Test `TaskRun`/`PipelineRun` of `Task`/`Pipeline`s.

## Idea

Automatically generate a pipeline to make sure the pipeline works as expected.

I would like to do a test on my pipeline.

```yaml
apiVersion: tekton.ornew.io/v1alpha1
kind: TaskTest
metadata:
  name: my-task-test
spec:
  # target task
  taskRef:
    name: my-task

  # validation that can be parsed statically.
  checks: |
    error[msg] {
      input.metadata.labels.foo == "bar"
      input.spec.steps == ["say", "err", "..."]
    }

  # test cases run as TaskRun
  tests:
    cases:
      - annotations:
          test_case: "a"
        params:
          - name: input
            value: foo
    afterAll: |
      ...
    afterEach: |
      error[msg] {
        some c
        tr := input.status.taskRuns[_]
        tr.status.conditions[c].status == "True"
        msg := sprintf("Task %s failed: %s", tr.name, c.message)
      }
    steps:
      - name: say
        stub:
          results:
            - name: output
              value: foo-bar
        after: |
          error[msg] {
            input.results.output == sprintf("%s-bar", input.params.input[_].value)
          }

      - name: err
        stub:
          exitCode: 1
        after: |
          error[msg] {
            input.current.status.terminated.exitCode == 1
          }

      - name: git-clone
        afterRun:
          - description: >-
              repos should be cloned to <workspace>/<results.output> dir.
            script: |-
              set -ex
              path=$(cat $(results.output.path))
              test -d $(workspace.repo.path)/${path}/.git
```

will generate to:

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: my-task-test-run
  # ...
spec:
  pipelineSpec:
    tasks:
      - name: first-1
        params:
          - name: input
            value: hi
        taskSpec:
          # copy the original task and patch it.
          params:
            - name: input
          results:
            - name: output
          steps:
            - # replace `say` step as a stub.
              name: say
              script: |
                echo "foo-bar" > $(results.output.path)
            - name: say-after
              script: |
                # check by rego
            - # replace and emit error
              name: err
              onError: continue  # onError value follows original task.
              script: |
                exit 1
            - name: git-clone-after
              script: |
                # check by rego
            - name: git-clone
              # this is original?
            - name: git-clone-after
              script: |
                # check byh rego
            - name: keep
              # not changed. run the original step.
      - name: after-first-1
        runAfter: [first-1]
        # ...
```

Features:

- Validation Task
- Test Double of `Task`

```
# pipeline test image
```
