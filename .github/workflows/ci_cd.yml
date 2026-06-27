name: Build and Push to AWS ECR

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

env:
  AWS_REGION: ${{ secrets.AWS_REGION }}
  ECR_REPOSITORY: ${{ secrets.ECR_REPOSITORY }}
  IMAGE_TAG: ${{ github.sha }}

jobs:
  build-and-push:
    name: Build and Push Docker Image
    runs-on: ubuntu-latest

    permissions:
      id-token: write
      contents: read

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Log in to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build Docker image
        run: |
          docker build \
            --tag ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ env.IMAGE_TAG }} \
            --tag ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:latest \
            .

      - name: Push Docker image to ECR
        run: |
          IMAGE_URI=${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}
          docker push "$IMAGE_URI:${{ env.IMAGE_TAG }}"
          docker push "$IMAGE_URI:latest"

      - name: Output image URI
        run: |
          echo "Image pushed to: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ env.IMAGE_TAG }}"

  trivy-scan:
    name: Scan Image with Trivy
    runs-on: ubuntu-latest
    needs: build-and-push

    permissions:
      id-token: write
      contents: read
      security-events: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Log in to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Run Trivy vulnerability scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ steps.login-ecr.outputs.registry }}/${{ secrets.ECR_REPOSITORY }}:${{ env.IMAGE_TAG }}
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH
          exit-code: "0"
          ignore-unfixed: true

      - name: Upload Trivy results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy-results.sarif

  deploy:
    name: Deploy to ECS
    runs-on: ubuntu-latest
    needs: trivy-scan
    environment: production

    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Log in to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Deploy to ECS
        run: |
           aws ecs update-service \
            --cluster ${{ secrets.ECS_CLUSTER_NAME }} \
            --service ${{ secrets.ECS_SERVICE_NAME }} \
            --force-new-deployment
