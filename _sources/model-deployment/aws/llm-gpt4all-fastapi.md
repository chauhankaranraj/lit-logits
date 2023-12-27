# Large Language Model (LLM) via FastAPI

## Package the Service into a Container Image

To ensure consistency, reproducibility, and portability, we will package our service into a conatiner image.

1. Create a `poetry` project with the following dependencies.
    ```
    mkdir llm-gpt4all-fastapi
    cd llm-gpt4all-fastapi
    pyenv local 3.10.13
    poetry init    # Set compatible python version as 3.10.13 here
    poetry add gpt4all fastapi uvicorn
    ```

2. Export the locked dependencies as a `requirements.txt` file. This is so that we can directly `pip install` packages within the container image rather than having to install `poetry` to maintain Python environment.
    ```
    poetry export -f requirements.txt --output requirements.txt
    ```

3. Create a python script called `app.py` with the following content to serve requests.
    ```
    import uvicorn
    from gpt4all import GPT4All
    from pydantic import BaseModel
    from fastapi import FastAPI
    from fastapi.responses import PlainTextResponse

    # init app
    app = FastAPI()

    # load model
    print('\n--> LOADING MODEL')
    model = GPT4All("mistral-7b-instruct-v0.1.Q4_0.gguf")

    # define request signature
    class Request(BaseModel):
        text: str
        # defaults from gpt4all docs
        temp: float = 0.7
        max_tokens: int = 200

    @app.get("/generate", response_class=PlainTextResponse)
    async def generate(req: Request):
        resp = model.generate(req.text, max_tokens=req.max_tokens, temp=req.temp)
        return resp

    if __name__ == "__main__":
        uvicorn.run(app, host="0.0.0.0", port=8080)
    ```

4. Add a Dockerfile.
    ```
    # Use an official Python runtime as a parent image
    FROM python:3.10.13-slim

    # Set the working directory in the container
    WORKDIR /usr/src/app

    # Copy the current directory contents into the container at /usr/src/app
    COPY app.py requirements.txt .

    # Install any needed packages specified in requirements.txt
    RUN pip install --no-cache-dir -r requirements.txt

    # Make port 8080 available to the world outside this container
    EXPOSE 8080

    # Run uvicorn server
    CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8080"]
    ```

5. Build the container image.
    ```
    dk build -t llm-gpt4all-fastapi-demo .
    ```

6. Test that the container runs as expected. In one terminal (or tmux pane), run the following to start the server
    ```
    dk run -p 8080:8080 llm-gpt4all-fastapi:latest
    ```
    Then, in another terminal (or tmux pane), send a GET request to the server
    ```
    curl -X 'GET' 'http://localhost:8080/generate' \
        -H 'accept: application/json' \
        -H 'Content-Type: application/json' \
        -d '{"text": "List all shapes that have 4 sides", "temp": 0.5, "max_tokens": 100}'
    ```

7. If the above produces an output (hopefully a coherent one, but not necessarily so :)), upload the container image to an Amazon ECR repository. To create an ECR repo and upload the image, run the following commands.
    ```
    aws ecr create-repository --repository-name llm-gpt4all-fastapi-demo
    aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.us-east-1.amazonaws.com
    dk tag llm-gpt4all-fastapi-demo:latest <aws_account_id>.dkr.ecr.us-east-1.amazonaws.com/llm-gpt4all-fastapi-demo:latest
    dk push <aws_account_id>.dkr.ecr.us-east-1.amazonaws.com/llm-gpt4all-fastapi-demo:latest
    ```

## Deploy the Service on EC2

1. Follow steps [here](../../personal-workspace/remote-desktop-instance#aws) to launch an EC2 instance and SSH into it. NOTE: this time you can choose a bigger instance type (e.g. `c5a.4xlarge`) since LLMs require a decent amount of resources.

2. Follow steps [here](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html#getting-started-install-instructions) to install `aws`.

3. Follow steps [here](../../personal-workspace/working-environment#docker) to install `docker`. You'll need to logout and SSH back in for the user group change to take effect.

4. Run a container using the image uploaded to the ECR repo in the above section.
    ```
    export AWS_ACCESS_KEY_ID=abcd
    export AWS_SECRET_ACCESS_KEY=efgh
    aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.us-east-1.amazonaws.com
    docker run -p 8080:8080 825685039652.dkr.ecr.us-east-1.amazonaws.com/llm-gpt4all-fastapi-demo:latest
    ```

5. Update security group to allow access (the following only allows access to your current device).
    ```
    aws ec2 authorize-security-group-ingress --group-name $GRPNAME --protocol tcp --port 8080 --cidr $PUBLICIP/32
    ```

6. Test the url from an allow-listed device.
    ```
    INSTIP=$(aws ec2 describe-instances --instance-ids $INSTID --query "Reservations[*].Instances[*].PublicIpAddress" --output text)
    curl -X 'GET' http://$INSTIP:8080/generate \
        -H 'accept: application/json' \
        -H 'Content-Type: application/json' \
        -d '{"text": "List all shapes that have 4 sides", "temp": 0.5, "max_tokens": 100}'
    ```

7. Let's try to get a more useful output, say, a "medium" level SQL interview question :)
    ```
    INSTIP=$(aws ec2 describe-instances --instance-ids $INSTID --query "Reservations[*].Instances[*].PublicIpAddress" --output text)
    curl -X 'GET' http://$INSTIP:8080/generate \
        -H 'accept: application/json' \
        -H 'Content-Type: application/json' \
        -d '{"text": "The table `employees` has columns `id` representing employee unique identifier, `first_name` representing first name, `last_name` representing last name, `salary` representing annual salary in USD, and `department_id` representing department unique identifier where the employee works. The table `departments` has columns `id` representing department unique identifier and `name` representing department name. Write a PrestoSQL query to extract top 3 departments with at least ten employees and rank them according to the percentage of their employees making over 100K in salary. The output should have columns `percentage_over_100k` showing percent of employees making over 100000, `department_name` showing department name, and `number_of_employees` showing total number of employees in department", "temp": 0.2, "max_tokens": 512}'
    ```
    This produces the following output
    ```
    ## Answer:
    WITH ranked_dept AS (
        SELECT d.id, d.name, COUNT(*) as num_employees,
            SUM(CASE WHEN e.salary > 100000 THEN 1 ELSE 0 END) / COUNT(*) * 100 AS percentage_over_100k
        FROM departments d
        JOIN employees e ON d.id = e.department_id
        GROUP BY d.id, d.name
    ),
    top3_dept AS (
        SELECT * FROM ranked_dept
        WHERE num_employees >= 10
        ORDER BY percentage_over_100k DESC
        LIMIT 3
    )
    SELECT top3_dept.name, top3_dept.num_employees, top3_dept.percentage_over_100k
    FROM top3_dept;
    ```
    Would you look at that, we have a data scientist assistant at our disposal now!
