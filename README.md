[![CI](https://github.com/halfmoonliu/SongRecommendation/actions/workflows/cicd.yml/badge.svg)](https://github.com/halfmoonliu/SongRecommendation/actions/workflows/cicd.yml)

# Using User Prompt for Song Recommendation

## Objectives

This repository is dedicated to the development of a **voice-triggered song recommendation application** (App). Our primary aim with this project is to empower users to effortlessly convey their current emotions or situations using **voice** commands, such as "I am waiting for my plane" or "I am feeling down today." Subsequently, the App will intelligently recommend and play a suitable song from Spotify that aligns with the user's mood. Furthermore, users have the option to choose a specific mood from a predefined set of six moods, including happy, sad, energetic, calm, anxious, and chill, via a **user-friendly pull-down menu** (Demo Video).

A crucial aspect of this project involves leveraging a cloud-based environment, specifically **Azure Databricks**, to deploy our functional web microservice. This approach ensures that our service can operate seamlessly, regardless of the number of concurrent users, and guarantees a high level of reliability. To enhance song recommendations, we have integrated the **OpenAI API**, enabling us to curate a list of recommended songs based on users' voice inputs.

In order to provide users with a diverse selection of songs for each mood category, we have meticulously generated a song dataset. This dataset encompasses six distinct sentiment categories: Happy, Sad, Energetic, Calm, Anxious, and Chill. Each sentiment category contains a curated list of 50 songs, complete with song titles, artist names, and mood labels. To ensure efficient storage and reliability, we initially created a CSV file and then stored it in a **Delta Lake** table within Databricks. This storage approach not only guarantees high performance but also offers scalability and flexibility to accommodate future needs.

The following sections provide a detailed **walkthrough** of the App's functionality and usage.


## Project Structural Diagram
![Final-26](https://github.com/halfmoonliu/SongRecommendation/assets/141781876/f087bfcc-f0e2-4f93-815a-b0ce00bce2dd)




## Key Components

- ``Microservice``: Our user-facing microservice, implemented in `app.py`, utilizes the Spotify API and OpenAI GPT-3 API. Auto-scaling is facilitated through Azure App Service for improved availability.
  
- ``Data Engineering Pipeline``:  This pipeline is encapsulated in ``mylib/extract.py`` and ``mylib/transform_load.py``. It demonstrates the creation of music data files and their upload to Azure Databricks File System (DBFS) via Azure Databricks REST API. Additionally, we showcase Delta Lake, chosen for its ACID (Atomicity, Consistency, Isolation, Durability) transaction support, ensuring robust performance.

- ``Container Configuration``: The Dockerfile provides clear instructions for creating a container that seamlessly fits the microservice's runtime environment.

- ``Load Test``: By harnessing the power of the Locust library, we perform thorough local load testing on the microservice. This approach allows us to assess the system's performance under high load conditions, ensuring that we can confidently handle anticipated loads and traffic without experiencing slowdowns or crashes.
  
- ``Github Configurations``: Our GitHub settings and parameters optimize collaboration and version control for the project.
  
- ``Infrastructure as Code (IaC)``: We employ Infrastructure as Code (IaC) principles to define, manage, and provision our project's infrastructure, promoting consistency and efficiency throughout its lifecycle.

## Data Engineering Pipeline 
![image](https://github.com/halfmoonliu/SongRecommendation/assets/141780408/24042efa-b022-4c07-bade-d776d15aa2cc)
```bash
# Load environment variables for authentication
load_dotenv()
server_h = os.getenv("SERVER_HOSTNAME")
access_token = os.getenv("ACCESS_TOKEN")
FILESTORE_PATH = "dbfs:/FileStore/Final"

# Importing SparkSession from the pyspark.sql library
from pyspark.sql import SparkSession
from pyspark.sql.functions import monotonically_increasing_id

def load(dataset="dbfs:/FileStore/Final/songs.csv"):
    spark = SparkSession.builder.appName("Read CSV").getOrCreate()
    # load csv and transform it by inferring schema 
    songs_df = spark.read.csv(dataset, header=True, inferSchema=True)

    # transform into a delta lakes table and store it 
    songs_df.write.format("delta").mode("overwrite").saveAsTable("songs_delta")
```
we initially established a secure connection to the Databricks environment by leveraging environment variables for authentication, specifically the SERVER_HOSTNAME and ACCESS_TOKEN. This connection allowed us to seamlessly integrate our GitHub repository with the Databricks Workspace repository.

Next, we initiated the data transformation process by converting the music_data.csv file into a Spark DataFrame. This DataFrame was then transformed into a Delta Lake Table, offering a structured and optimized storage format within the Databricks environment.

To ensure the continuous synchronization of our data, we implemented a dedicated job responsible for constructing the Databricks ETL (Extract, Transform, Load) pipeline. This ensures that any changes made to the CSV data with every GitHub push are automatically reflected in our Databricks Delta Lake, guaranteeing up-to-date and accurate data for analysis.

## Container Configuration
A Dockerfile within a GitHub repository plays a pivotal role in the deployment of a web application on Azure via the Azure Container Registry (ACR). Essentially, a Dockerfile is a scripted compilation of all the commands that would typically be issued in the command line to create a Docker image. Within the GitHub ecosystem, this file serves as a foundational script for constructing a Docker container, guaranteeing that the web application's environment is uniform, replicable, and subject to version control.
```python
# Use an official Python runtime as a parent image
FROM python:3.9.7-slim

# Set the working directory in the container
WORKDIR /app

# Copy the current directory contents into the container at /app
COPY . /app

# Install any needed packages specified in requirements.txt
RUN pip install --no-cache-dir -r requirements.txt

# Make port 8502 available to the world outside this container
EXPOSE 8502

# Define environment variable
ENV NAME World

# Run app.py when the container launches
CMD ["streamlit", "run", "app.py", "--server.port", "8502"]
```
The deployment journey of a web app on Azure using ACR commences with the Dockerfile situated in the GitHub repository. This critical file delineates the application's environment, specifying aspects such as the base operating system (e.g., Ubuntu or Alpine), necessary dependencies, environment variables, and the commands needed to run the app. After finalizing and committing the Dockerfile to your GitHub repository, Docker is employed to build an image from this file. This resultant image is a comprehensive package containing all the requisite components to run your application, neatly bundled into a portable, self-contained container. The following is a brief walkthrough from building docker image to push this image to Azure Container Registry. 

- 1. login to Azure using the Azure CLI. This command will open a web page where you can enter your Azure credentials.
```bash
az login
```
- 2. create or use existing Azure Container Registry.
```bash
az acr create --resource-group data_engineering --name songregistry --sku Basic
```


The subsequent phase involves pushing this constructed image to the **Azure Container Registry**. ACR is a dedicated and private repository designed to host container images, seamlessly integrating with existing container development and deployment tools, while also offering a secure storage solution for our images. Once our image is securely housed in ACR, we can leverage Azure Web App for Containers for the deployment of your web application. This Azure service facilitates the deployment of your containerized application, complete with bespoke configurations, and provides the flexibility to scale as per our operational requirements. Azure efficiently retrieves the image from ACR and executes it within a container, ensuring that the runtime environment mirrors precisely what was outlined in the Dockerfile.[show the images in azure that assures successful deployment of docker container registry]
<img width="723" alt="Screenshot 2023-12-10 at 11 51 05" src="https://github.com/halfmoonliu/SongRecommendation/assets/141781876/2d5b56ac-ac00-4c80-bc8b-7db9778bc093">


This streamlined process exemplifies the efficacy of combining Docker with Azure's capabilities, ensuring consistent application behavior across different stages of development, testing, and production, and capitalizing on Azure's scalability and security features, all while maintaining the agility and control afforded by containerization. Finally, the app was successfully deployed via **Azure App Service**. Detailed information of web app functionality on Azure can be found and diagnostic through logs. [image of web deployment and log]
![Uploading Screenshot 2023-12-10 at 11.53.25.png…]()
<img width="1003" alt="Screenshot 2023-12-10 at 11 52 53" src="https://github.com/halfmoonliu/SongRecommendation/assets/141781876/e58702a4-aa59-44dd-a8d4-67a308e257aa">

## YC Liu
1. Main Functions associated libraries .
  <br>a. _main.py_: the main function for the
  <br>b. _./libraries_:
      <br>i.   _01_speech_to_text.py: **tranforms user's input** voice into **text**. 
      <br>ii.  _02_gpt_prompt.py: uses transfomed user input to **prompt chat GPT** and **returns song recommendations received**.
      <br>iii.  _03_spotify_functionality.py: takes a **song name** and an **artist name** and feed them to **Spotify API** and output **spotify soundtrack playing URL**.
      <br>iv. _04_query.py: **querying song names and artist names** based on **user input mood**.
      <br>v. _05_parser.py: output **one song name** and the corresponding **artist name** by parsing chat GPT's response.
      <br>vi.  _songDB.py_: contains functions for **building a song database**.

3. **Github actions setup for continuous integration**
      <br>c. _.github/workflows/main.yml_: Quality control actions are triggered when pushed/ pulled to main branch. After setting up the environment, actions of **installing packages**, **linting**, **testing**, **formatting** would be executed in order (specified in Makefile). 

4. **Other files for development environment settings**
      <br>d. _.gitignore_: specify file names to ignore.
      <br>e. _requirements.txt_: list required packages for the project.

5. **Description of the project**
   <br>f. _README.md_: THIS FILE, explaining the purpose and structure of the directory, with example output and code snippets.

## Contributors 
- [Afraa Noureen](https://github.com/afraa-n)
- [Bob Zhang](https://github.com/BobZhang26)
- [Jiwon Shin](https://github.com/jiwonny29)
- [Yun-Chung (Murphy) Liu](https://github.com/halfmoonliu)
