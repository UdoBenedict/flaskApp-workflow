# Learning CI/CD: Building My First Automated Pipeline From Scratch

This feels surreal, executing my first CI/CD pipeline & documenting the process. For the longest, I have been in that cycle of starting a project, deploying manually, and then promise myself that I will automate the flow in the next project. 

This September, I finally stopped procrastinating and built my first proper CI/CD pipeline for a Flask app. And you know what? It was actually enjoyable once I got started. I made few errors as expected but in all, a lot to learn in this project. I created a pipeline that automatically:

- log into my Docker Hub account.
- builds Docker images when I push code.
- pushes images to Docker Hub with version tags.
- deploys to my server without any manual steps.
- runs health checks to confirm everything works.
- cleans up old images to save space.

CI/CD is all new territory to me and so I ensured I kept things as simple as possible.

## Why I Chose GitHub Actions

I am aware there are plenty of CI/CD tools — Jenkins, GitLab CI, CircleCI etc, which can cause prove difficult to choose from. I went with GitHub Actions because it integrates seamlessly with where my code already lives.

I didn’t want to juggle multiple platforms or disrupt my workflow. GitHub Actions works directly with my repositories, doesn’t require maintaining another service, and honestly, has pretty good documentation. It’s CI/CD without having to leave my development environment.

## Tech Stack

`GitHub Actions` - `Docker` - `AWS` - `Python`

## Key Capabilities

A simple flask app that returns "Hello, World!" on root path and returns "Health check ok!" when `/health` endpoint is reached.

## Deployment Guide

### 1. Set up GitHub Secrets

All sensitive information lives securely in GitHub Secrets:
1. In the GitHub repository → Settings → Secrets and variables → Actions

Add these secrets:
— `DOCKERHUB_USERNAME` (my Docker Hub username)
— `DOCKERHUB_TOKEN` (generate in Docker Hub Account Settings → Security)
— `SERVER_HOST` (AWS EC2 public or elastic IP or hostname)
— `SERVER_USER` (SSH username)
— `SERVER_SSH_KEY` (private SSH key for the server)

### 2. Login to Docker Hub

- Publish the image for deployment.
- Log in with secrets.

### 3. Flask Application to Dockerfile + Docker Image Build & Push

- Push both version tags & git commit SHA.
- Use `docker build -t $IMAGE_NAME:$GITHUB_SHA .`.

### 4. Deploy to AWS EC2

- SSH into your EC2 instance and update the running container.
- Stop old container if running.
- Pull new image from Docker Hub.

### 5. Run Health Checks

- Verify the app is serving correctly.
```
Using curl http://$AWS_HOST/health
```
- Fail workflow if response is not `200 OK.

### 6. Clean Up Old Images

- Save disk space on the server.
- Run `docker image prune -af`


This pipeline ensures:
1. Triggers on main pushes.
2. Securely logs into Docker Hub using GitHub Secrets
3. Docker Images are built, tagged with two image tags: `v1` and the `commit SHA` and pushed to Docker Hub.
4. AWS server pulls and runs the latest container.
5. Health checks confirm the app is live.
6. Old images are cleaned up to save space.

## Errors I Encountered + Lessons learnt

### 1. Adding the version tag to the IMAGE_NAME

That error occured because the Docker tag is malformed, i.e. two colons in the tag string `"***/demo-pipeline-flask-app:v1:03aa093581defe6885374d55f7122f52bffabc60"`. 
```
Run IMAGE_NAME=***/demo-pipeline-flask-app:v1
ERROR: failed to build: invalid tag "***/demo-pipeline-flask-app:v1:03aa093581defe6885374d55f7122f52bffabc60": invalid reference format
Error: Process completed with exit code 1.
```

#### Lesson learnt:
1. Do not include the version tag `latest` or `v1` in the `IMAGE_NAME` variable. Instead, include it when calling the variable during build or push. See example below:

```
IMAGE_NAME=${{ secrets.DOCKERHUB_USERNAME }}/demo-pipeline-flask-app
docker build -t $IMAGE_NAME:v1 -t $IMAGE_NAME:${{ github.sha }} .
docker push $IMAGE_NAME:v1
docker push $IMAGE_NAME:${{ github.sha }}
```
2. Ensure that the server permits SSH on port 22.
3. Ensure Docker is installed.
4. Add user to the docker group (so no need sudo for every command). Then log out and back in (or run newgrp docker) so the group membership takes effect.
```
sudo usermod -aG docker ec2-user
```
5. Ensure AWS EC2 security group allows inbound TCP traffic on port 5000, as the container is listening on that port. If port 5000 isn’t open, curl from outside will fail even though Gunicorn is listening inside.

## What’s Next: Where I Want to Take This
1. Pull Request Integration
Currently, the pipeline only runs on `main`. I want to add workflows that trigger on pull requests — maybe running tests or building preview environments. This would help catch issues before they reach production.

2. Adding a Database   
My current app is stateless. To make this more realistic, I’d like to add PostgreSQL and learn how to handle:
- Database migrations as part of deployments
- Environment variables and secrets management
- Connection pooling and persistence

3. Testing Pipeline
I’m planning to integrate pytest to run tests automatically. This is new territory for me, but it seems like a logical next step.

Future Improvements
Once I’m more comfortable with the basics:
- Staging environment: Separate from production for testing
- Database migration automation: I’ll need to learn how to do this properly
- Monitoring integration: Something like Sentry for error tracking.
- Multi-container setup: Maybe add Redis or Nginx once I understand the single-container setup better.


