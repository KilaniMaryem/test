## Project Overview
The project involves establishing an MLOps pipeline for training models for multi-label classification of images extracted from DICOM files. The data pipeline is designed to work with DICOM formatted data, handling their loading, image and metadata extraction from DICOM files, storing them in PostgreSQL and MinIO. Additionally, the data pipeline incorporates a quality check process for the images .



## DAGS Description

### ExtractDAG
1. create_cnxs_task: Creates or updates postgres and minio connections required for the pipeline's interaction with external services.
2. branch_operator: This operator determines whether new folders have been added or if there are no updates. It directs the execution flow accordingly.
3. no_updates_task: indicates that no unprocessed folders were found during the execution of the pipeline.
4. new_folders_added_task: indicates the presence of new folders to be processed.
5. download_dicom_files_task: downloads DICOM files from the NAS server, preparing them for further processing.
6. anonymize_data_task: anonymizes DICOM data by removing sensitive information.
7. extract_imgs_task: extracts images from DICOM files and saves them as PNG images.
8. extract_metadata_task: Extracts metadata from DICOM files and saves it in JSON format.
9. rename_files_task: renames the DICOM files in the dicomFiles folder to include a "processed" tag. This modification is a crucial indicator that the files have undergone image and metadata extraction. By adding the "processed" tag, the files are distinctly marked as completed, preventing redundant processing during subsequent DAG runs.
10. rename_NASfolders_task: Renames folders on the NAS server to indicate that they have been processed, preventing redundant processing during subsequent DAG runs.

### StoreDAG
1. metadata_sensor: This sensor task monitors the completion of the extract_metadata_JSON task from the ExtractDag. It reschedules itself until the metadata extraction task is successfully completed. This ensures that metadata extraction is finished before proceeding.
2. save_to_postgres_task: This task is crucial for storing metadata in a PostgreSQL table. It creates a table if it doesn't exist, processes metadata files, and inserts metadata into the table.
3. rename_metadata_files_task: This task renames processed metadata files in the dicomMetadata folder by appending a "processed" tag to their filenames. This is done to mark files as processed, preventing duplication in future DAG runs.
4. imgs_sensor: Similar to the metadata_sensor, this sensor task monitors the completion of the extract_imgs task from the ExtractDag. It waits for image extraction to complete before proceeding.
5. upload_images_task: This task handles the uploading of processed images to a MinIO server. Images are organized into buckets, and new buckets are created as needed. The bucket naming convention includes a prefix and a timestamp to prevent bucket overflows. Each bucket can store up to 400 images.
6. rename_png_images_task: This task renames processed image files in the dicomImages folder by appending a "processed" tag to their filenames. This renaming signifies that the images have been processed and prevents further processing.

### PreprocessingDAG
1. minio_sensor: This sensor task ensures that the preceding rename_png_images task from the StoreDag is completed before starting the preprocessing tasks. This ensures that the previous dag finished before proceeding with quality checks.
2. check_black_images_in_buckets: This task scans through buckets containing images and checks if any images are excessively dark. It identifies images with low average intensity, and if found, it moves them to a "suboptimal" bucket.
3. quality_check_task: This task performs a quality check on images using an SVM model (you can find it in the model folder).If an image is predicted to be of low quality, it's moved to the "suboptimal" bucket. The names of buckets that have undergone the quality check process are saved (to prevent repeated checks) in a JSON file located in the "requirements" folder. 

### AnnotationsDAG
1. branch_operator: A BranchPythonOperator task that determines whether there are new annotations files in the 'annotations' bucket.
2. no_annotations_updates_task: Prints a message indicating that there are no unprocessed annotations files.
3. new_annotations_added_task: Prints a message indicating that new annotations files have been added.
4. download_annotations_files_task: Downloads unprocessed annotations files from the MinIO bucket into the annotations folder.
5. update_postgres_task: Updates the PostgreSQL table with annotations data from the downloaded annotations files, and renames the annotations files by adding "processed" to their filename.
6. rename_minio_annotations_files_task: Renames processed annotations files in the MinIO bucket.

### DAGS Dependencies
The dependencies and synchronization between the first three DAGs (ExtractDag, StoreDag, and PreprocessingDag) are ensured through the use of ExternalTaskSensors. These sensors monitor the completion status of specific tasks within each of the mentioned DAGs before proceeding with their own tasks.

On the other hand, the AnnotationsDag operates in a slightly different manner. Unlike the other DAGs, it is not automatically triggered by the completion of tasks in preceding DAGs. Instead, it can be manually triggered when needed. This means that the AnnotationsDag provides flexibility and control to users, allowing them to initiate the DAG's execution as required, independently of the progress of other workflows.

## Installation

### Docker images
1. extended_airflow:1.0 : built using the Dockerfile with the command: "docker build -t extended_airflow:1.0 ."
2. minio/minio:latest
3. postgres:13
### Required librairies
The required libraries are listed in the 'requirements.txt' file in the requirements folder.
### Steps
1. 
```bash
   docker build -t extended_airflow:1.0 .
```
2. 
```bash
 docker compose up -d
 ```

## Folders
1. downloads: This folder serves as a temporary storage location for DICOM files before they are transferred to the dicomFiles directory. This arrangement is necessitated by the implementation of the Synology API's get_file function.
2. dicomFiles: contains the downloaded dicom files are stored for further processing.
3. dicomImages: contains the extracted images from the dicom files in a PNG format.
4. dicomMetadata: contains the extracted metadata from the dicom files in a JSON format.
5. requirements: contains the 'requirements.txt', the 'config.json' and the 'checked_buckets.json' files.
6. dags: contains teh dags python files.
7. annotations: contains the downloaded annotations files from the 'annotations' bucket.
8. model: contains the SVM quality check (good/bad) model in a pkl format. The model is then loaded in the task with the joblib library.
9. tests: contains a dags test file. The airflow-init service will run the file.

## Configuration
To configure the pipeline for your environment, you need to provide the necessary credentials for PostgreSQL, MinIO, and the NAS server.
These credentials are in the 'config.json' file in the requirements folder.