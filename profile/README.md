# surfsight

Computer Vision Analytics for Surf-spots

## Overview

This repo includes the components that make up the [SurfSight.ai](https://surfline.ai) Computer Vision Analytics (CVA) platform: 

![CVA System](https://github.com/SurfSight/blob/master/.github/figures/CVA_System.png "CVA System")

The primary purpose of the platform is to extract information from surf-spot webcams using computer vision, aggregate valuable insights from the extracted information, and expose the insights to the end user in a digestible and actionable manner. The platform consists of 5 primary components:

- **Training**: the machine learning elements (data, code, configurations, etc.) necessary for training computer vision models on surf-data (counting surfers in the water, classifying visibility, segmenting waves, etc.)
- **Endpoint**: the API endpoints hosting trained models used to extract information from webcam frames
- **Backend Client**: the orchestrator and scheduler responsible for monitoring the webcams, performing inference, and recording insights in database tables 
- **Spot Data**: the Django ORM layer that sits atop the database tables and defines the functionality for interacting with the computer vision generated insights
- **Frontend**: a simple web application that visualizes insights from the database tables on a spot-by-spot basis

More detail on the individual components below.

### Training

Training is conducted using the [TensorFlow Object Detection API](https://github.com/tensorflow/models/tree/master/research/object_detection) on TPUs. The entire configuration of a single training run is defined in a config file, and individual training runs are conducted as jobs on the Google Cloud AI Platform using a custom Docker image. This combination of the TF2 OD API and AI Platform allows for quick training iteration. The only two changing factors are the dataset and the training configuration, both of which are defined in the config file. Training on new datasets and/or with different training parameters is a simple as uploading a new config file to a Google Storage bucket and submitting a job with the new configuration to the AI Platform.

![Training Pipeline](https://github.com/SurfSight/blob/master/.github/figures/Training_Pipeline.png "Training Pipeline")

### Endpoint

The endpoint is built using FastAPI and deployed using Uvicorn in a Docker image. Due to the fluctuating nature of the inference load Kubernetes is used to manage and scale endpoint pod replicas. The use of Kubernetes also facilitates canary deployment of new models, resource scaling, and the ability to dynamically increase the functionality of the platform by gradually deploying new features (model endpoints) without service interruptions.

### Backend Client

The backend client schedules and orchestrates the model inference, insight processing, and writing insights to database tables. AirFlow DAGs are used to schedule the various inference and computation tasks at fixed intervals. A list of active spot objects is passed around between the tasks, and the individual task computations are executed with Python operators that call functions on the active spot instances. The functions are defined as part of the Django ORM spot model.

### Spot Data

Django ORM allows for easy interaction with the insight data. The heavy lifting is done behind the scenes by defining the desired data functionality as Django models. As new insight features are added to the platform the functionality of the models can be increased by adding attributes and functions to the Django spot model. The use of Django ORM allows for separation of responsibilities: AirFlow is responsible for the when something is done, Django models are responsible for how something is done.

### API

A standard REST API sitting on top of the postgres instances. This allows for easy insight into the data aggregation, and simplify the integration of subsequent applications. Token AUTH is in affect. 

### Frontend

The front end is a basic web application that consumes data from the spot tables and renders it in a simple and useful form. As the functionality of the platform is increased (more models are added for additional insights and more data is written to the database tables) new features can be added to the frontend to visualize the new insights.

![Frontend UI](https://github.com/SurfSight/blob/master/.github/figures/Frontend_UI.png "Frontend UI")

## Next Steps

The platform is still in the early phases and requires more structure before it is production ready. Some of the next steps are:

- Model iteration: adjust the training configuration of the surf-count model to focus on small-object performance (anchor box adjustments, non-max suppression fine-tuning, etc.)
- Data iteration: increase the size and scope of the training data (with an emphasis on spots/environments where the model preforms poorly), fix bad annotations (inconsistent boxes, occluded surfers, etc.), experiment with data augmentation
- Endpoint efficiency: increase inference speed by exporting the model in a more endpoint-friendly format (quantized, frozen graph, TFLight, etc.) and transitioning from REST to gRPC protocol, and decrease size of endpoint container for resource efficiency
- Continuous training: set up cloud triggers to monitor for new config files and automatically submit training jobs to the AI Platform
- Inference monitoring: set up qualitative and quantitative monitoring for model predictions (bounding-box heat maps and histograms, confidence heat maps and histograms, etc.)
- Testing: write testing functions for individual components to aid in the CI/CD process
- CI/CD: set up dev, test, and prod Github branches and use Jenkins/Spinnaker to automate integration and deployment
- Wave Segmentation: Apple Watch app that vibrates when a set is coming through
