# ECR Image Scan Github Action

Scan an image uploaded to ECR and fail if vulnerabilities are found.

## Quick Start

```yaml
      - name: Scan Docker image
        id: docker-scan
        uses: alexjurkiewicz/ecr-scan-image@v1.2.0
        with:
          repository: myorg/myimage
          tag: v1.2.3
          fail_threshold: high
```

## Inputs

| Input  | Required? | Description |
| ------ | --------- | ----------- |
| repository | :white_check_mark:  | ECR repository, eg myorg/myimage |
| tag    | :white_check_mark: | Image tag to scan |
| fail_threshold | | Fail if any vulnerabilities equal to or over this severity level are detected. Valid values: critical, high, medium, low, informational. Default value is high. |

## Outputs

| Output | Description |
| ------ | ----------- |
| total | Total number of vulnerabilities detected. |
| critical | Number of critical vulnerabilities detected. |
| high | Number of high vulnerabilities detected. |
| medium | Number of medium vulnerabilities detected. |
| low | Number of low vulnerabilities detected. |
| informational | Number of informational vulnerabilities detected. |
| unknown | Number of unknown vulnerabilities detected. |

## Example

This example builds a docker image, uploads it to AWS ECR, then scans it for vulnerabilities.

```yaml
on:
  # Trigger on any GitHub release.
  # If you want to trigger on tag creation, use `create`. However, this also
  # fires for branch creation events which will break this example workflow.
  - release
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-west-2
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1
      - name: Build & Push Docker image
        id: docker-build
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: myorg/myimage
          # Use the git tag as the image tag.
          # github.ref format is like `refs/tags/v0.0.1`, so we strip the the
          # `refs/tags/` prefix and export this for later use.
          IMAGE_TAG: ${{ github.ref }}
        run: |
          tag=${IMAGE_TAG##refs/tags/}
          echo "Tag is $tag"
          echo "::set-output name=tag::$tag"
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$tag .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$tag
      - name: Scan Docker image
        id: docker-scan
        uses: alexjurkiewicz/ecr-scan-image@v1.2.0
        with:
          repository: myorg/myimage
          tag: ${{ steps.docker-build.outputs.tag }}
          # fail_threshold: medium
      # Access scan results in later steps
      - run: echo "${{ steps.docker-scan.outputs.total }} total vulnerabilities."
```

## Development

This action is implemented as a Docker rather than a Javascript action because [that would require committing node\_modules to the repository](https://help.github.com/en/actions/building-actions/creating-a-javascript-action#commit-tag-and-push-your-action-to-github).

You can test the action by running it locally like so:

```sh
docker build -t ecr-scan-image:dev .
docker run -t \
  -e INPUT_REPOSITORY=myorg/myapp \
  -e INPUT_TAG=test-tag \
  -e INPUT_FAIL_THRESHOLD=critical \
  -e AWS_ACCESS_KEY_ID=xxx \
  -e AWS_SECRET_ACCESS_KEY=xxx \
  -e AWS_REGION=xxx \
  ecr-scan-image:dev
```
