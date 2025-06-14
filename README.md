# CLO835 Assignment 1 – Containerized WebApp and MySQL Deployment

This repository contains the application code and Docker configurations for a web-based employee database and a pre-seeded MySQL database. The goal is to build Docker images, push them to Amazon ECR, and run the containers on an EC2 instance provisioned via Terraform.

## File Structure

```text
├── app.py                # Flask web application
├── requirements.txt      # Python dependencies
├── templates/            # HTML templates
├── mysql.sql             # SQL schema and seed data
├── Dockerfile            # WebApp container image
├── Dockerfile_mysql      # MySQL custom image with preloaded database
├── .github/workflows/
│   └── deploy.yml        # GitHub Actions workflow for build and push
```

## WebApp Overview

The Flask application allows users to:

- Add new employees
- Fetch employee information by ID

The app reads from a MySQL database via environment variables:

- `DBHOST`
- `DBPORT`
- `DBUSER`
- `DBPWD`
- `DATABASE`

## MySQL Setup

The `Dockerfile_mysql`:

- Extends from `mysql:8.0`
- Copies `mysql.sql` to container
- Auto-initializes the employees database and table on container start

## Docker Build

To test locally:

```sh
# Build WebApp
docker build -t webapp -f Dockerfile .

# Build MySQL
docker build -t mysql -f Dockerfile_mysql .
```

## Local Test (Optional)

Create a custom Docker network and run both containers:

```sh
docker network create app-network

docker run -d --network app-network --name my_db -e MYSQL_ROOT_PASSWORD=pw mysql

docker run -d --network app-network -p 8081:8080 --name blue \
    -e APP_COLOR=blue \
    -e DBHOST=my_db -e DBPORT=3306 \
    -e DBUSER=root -e DBPWD=pw -e DATABASE=employees webapp
```

Then go to: [http://localhost:8081](http://localhost:8081)

## GitHub Actions Workflow

On every push to the main branch, GitHub Actions will:

- Build Docker images for both webapp and mysql
- Push them to their respective Amazon ECR repositories

You must set these secrets in your repo:

- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_SESSION_TOKEN`
- `AWS_REGION`

## ECR Repositories

- **webapp**: contains the Python app container
- **mysql**: contains the MySQL image with seeded data

## EC2 Container Deployment

After pushing to ECR, log in to your EC2 instance (provisioned via Terraform) and run:

### Post-Deployment Setup on EC2

To pull Docker images from Amazon ECR, configure AWS credentials and log into ECR on your EC2 instance.

1. **Create AWS credentials directory:**

   ```sh
   mkdir -p ~/.aws
   ```

2. **Edit credentials file:**

   ```sh
   nano ~/.aws/credentials
   ```

3. **Add your AWS credentials:**

   ```
   [default]
   aws_access_key_id = <YOUR_AWS_ACCESS_KEY_ID>
   aws_secret_access_key = <YOUR_AWS_SECRET_ACCESS_KEY>
   aws_session_token = <YOUR_AWS_SESSION_TOKEN> # Only if using temporary credentials
   ```

4. **Log in to Amazon ECR:**

   ```sh
   aws ecr get-login-password --region <AWS_REGION> | \
       sudo docker login --username AWS --password-stdin <AWS_ACCOUNT_ID>.dkr.ecr.<AWS_REGION>.amazonaws.com
   ```

5. **Pull Docker images:**

   ```sh
   docker pull <AWS_ACCOUNT_ID>.dkr.ecr.<AWS_REGION>.amazonaws.com/mysql:latest
   docker pull <AWS_ACCOUNT_ID>.dkr.ecr.<AWS_REGION>.amazonaws.com/webapp:latest
   ```

6. **Run containers:**

   ```sh
   docker network create app-network

   docker run -d --network app-network --name my_db -e MYSQL_ROOT_PASSWORD=pw \
       <AWS_ACCOUNT_ID>.dkr.ecr.<AWS_REGION>.amazonaws.com/mysql:latest

   docker run -d --network app-network -p 8081:8080 --name blue \
       -e APP_COLOR=blue \
       -e DBHOST=my_db -e DBPORT=3306 \
       -e DBUSER=root -e DBPWD=pw -e DATABASE=employees \
       <AWS_ACCOUNT_ID>.dkr.ecr.<AWS_REGION>.amazonaws.com/webapp:latest
   ```

Access the app via:  
`http://<EC2_PUBLIC_IP>:8081`

## Requirements

- Python 3
- Docker
- AWS CLI configured
- Terraform (for EC2 setup)

---

**Author:**  
Sandeep Subedi – CLO835 Assignment 1
