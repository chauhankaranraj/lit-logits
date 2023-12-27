# Classical ML Model via FastAPI

1. Follow steps here to launch an EC2 instance and SSH into it. NOTE: this time you can choose a smaller instance type (e.g. `c7a.medium`) since this is for demo purposes only.

2. Follow steps here to install `pyenv` and the Python version that your model was trained in (or is compatible with).

3. Create a project directory and set the Python version for it.
    ```
    mkdir app
    cd app
    pyenv local 3.10.13
    ```

4. Install the runtime dependencies.
    ```
    pip install boto3 fastapi uvicorn scikit-learn
    ```

5. Export AWS credentials as environment variables (or install aws locally and configure it).
    ```
    export AWS_ACCESS_KEY_ID=abcd
    export AWS_SECRET_ACCESS_KEY=efgh
    ```

6. Add the inference script.
    ```
    from fastapi import FastAPI
    from fastapi.responses import HTMLResponse
    from pydantic import BaseModel
    from typing import List
    import joblib
    import boto3


    # Init app
    app = FastAPI()

    # Download trained model from s3
    s3_client = boto3.client('s3')
    s3_client.download_file('lit-logits-demo', 'model-deployment/aws/sklearn/breast-cancer-predictor.joblib', 'model.joblib')
    model = joblib.load('model.joblib')


    # Human friendly home page
    @app.get("/", response_class=HTMLResponse)
    async def home():
        return """
        <html>
            <head>
                <title>Sample Scikit-Learn Predictor</title>
            </head>
            <body>
                <h1>Breast Cancer Predictor</h1>
                    <p>Please pass a data point to the /predict POST endpoint</p>
            </body>
        </html>
        """


    class BreastCancerDatapoint(BaseModel):
        # Single data point from the UCI breast cancer dataset
        features: List[float]


    @app.post('/predict')
    async def predict(data: BreastCancerDatapoint):
        prediction = model.predict([data.features])
        return {"prediction": prediction.tolist()}
    ```

6. Run the server.
    ```
    uvicorn main:app --host 0.0.0.0 --port 8080
    ```

7. Update security group to allow access (the following only allows access to your current device).
    ```
    aws ec2 authorize-security-group-ingress --group-name $GRPNAME --protocol tcp --port 8080 --cidr $PUBLICIP/32
    ```

8. Test the url from within the EC2 instance.
    ```
    curl -X 'POST'   'http://localhost:8080/predict'   -H 'accept: application/json'   -H 'Content-Type: application/json'   -d '{
    "features": [1.799e+01,1.038e+01,1.228e+02,1.001e+03,1.184e-01,2.776e-01,3.001e-01
        ,1.471e-01,2.419e-01,7.871e-02,1.095e+00,9.053e-01,8.589e+00,1.534e+02
        ,6.399e-03,4.904e-02,5.373e-02,1.587e-02,3.003e-02,6.193e-03,2.538e+01
        ,1.733e+01,1.846e+02,2.019e+03,1.622e-01,6.656e-01,7.119e-01,2.654e-01
        ,4.601e-01,1.189e-01]
    }'
    ```

9. Test the url from an allow-listed device.
    ```
    curl -X 'POST'   'http://3.237.36.158:8080/predict'   -H 'accept: application/json'   -H 'Content-Type: application/json'   -d '{
    "features": [1.799e+01,1.038e+01,1.228e+02,1.001e+03,1.184e-01,2.776e-01,3.001e-01
        ,1.471e-01,2.419e-01,7.871e-02,1.095e+00,9.053e-01,8.589e+00,1.534e+02
        ,6.399e-03,4.904e-02,5.373e-02,1.587e-02,3.003e-02,6.193e-03,2.538e+01
        ,1.733e+01,1.846e+02,2.019e+03,1.622e-01,6.656e-01,7.119e-01,2.654e-01
        ,4.601e-01,1.189e-01]
    }'
    {"prediction":[0]}%
    ```