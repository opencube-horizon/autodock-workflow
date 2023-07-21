# AutoDock-GPU workflow for Apache Airflow
A workflow for molecular docking using AutoDock-GPU. The workflow is implemented as a DAG, and can be run in Apache Airflow, on a Kubernetes cluster.

## Quickstart

The main DAG is contained in `autodock.py`, we also provide with the following folders:
- `docker/` contains the `Dockerfile`, along with bash scripts which are included in the image;
- `misc/` contains various configuration files, used for testing and development;
- `plot/` contains python scripts to create plots.

### Installation checklist
- [x]

## Setup & Installation
### 1. Kubernetes _PersistentVolume_ and _PersistentVolumeClaim_
The workflow relies on a specific PersitentVolumeClaim to be present on the Kubernetes cluster to store files during execution. In this step, we describe how to create a PersistentVolume, and a PersistentVolumeClaim attached to this volume.

**PersistentVolume**. Your Kubernetes cluster administrator provides you with the name of the PersistentVolume you need to use. However, if you manage your own Kubernetes cluster, you need to create a PersistentVolume yourself, we provide an example in `misc/persistentVolume.yaml`, that you can deploy using:

```
kubectl create -f misc/persistentVolume.yaml
```

In this example, and in the rest of this tutorial, the PersistentVolume is named `pv-autodock`. Please refer to Kubernetes documentation to learn more on PersistentVolume.

**PersistentVolumeClaim**. Once you know the name of the PersistentVolume (in this example `pv-autodock`), you need to create a PersistentVolumeClaim, containing information on the storage size, and referring to the underlying PersistentVolume. An example is provided in `misc/pvclaim.yaml`, you can create it using:

```
kubectl create -f misc/pvclaim.yaml -n airflow
```

In this example, and in the rest of this tutorial, the PersistentVolumeClaim is named `pvc-autodock`. Furthermore, it has the same namespace as the one under which your Apache Airflow setup is deployed, here we use `airflow`.

### 2. Building the Docker image
A Dockerfile is provided, along with scripts that will be included in the image, in the the `docker` folder.

To build and publish the image, when in the `docker` folder:
```
docker build -t gabinsc/autodock-gpu:1.5.3
docker push gabinsc/autodock-gpu:1.5.3
```

Please refer to Docker documentation for more details on building image, and publishing it. Make sure that you publish your image to a public Docker registry, or at least on which is accessible from your Apache Airflow setup.

### 3. Deploying and adapting the DAG
In order for the DAG to be executed in your specific environment, some adjusments are required.

1. Place the `autodock.py` file in the DAG folder of your Apache Airflow setup.
2. Adjust the following constants in autodock.py:
    - `IMAGE_NAME`: name of the image that will be used for the containers.
    - `PVC_NAME`: name of the PersistentVolumeClaim created in step 1, `pvc-autodock`.
3. Validate that you can see the DAG under the name `autodock` in the Apache Airflow UI. If not, DAG import errors are reported in the top of the UI.

The administrator of the Apache Airflow setup must create a pool named `gpu_pool`, which will group (and limit) execution of GPU tasks. You can create a pool in the Apache Airflow UI under `Admin > Pools`. A recommandation is to set the pool size to the desired maximum number of GPUs to be used in parallel.

## Run
Before you run the DAG, place your database of ligands, in `.sdf` format, in the root of the PersistentVolume you defined when configuring your Kubernetes cluster. For example, `sweetlead.sdf`.

Click on "Trigger DAG" in the Apache Airlfow UI to start the DAG with the default parameters. You can customize the DAG parameters to your needs by clicking "Trigger DAG w/ config":
- `pdbid`: PDBID of the protein you want to use as a receptor. Note that in the PDB database, this generally refers to a protein-ligand complex; the workflow automatically keeps the longest chain in the complex.
- `ligand_db`: the name of the ligand database, without the `.sdf` extension.
- `ligands_chunk_size`: bath size

## Result analysis and plotting
We provide python scripts to create readable Gantt charts, based on the workflow execution.
Data is directly fetched from Apache Airflow for a specific DAG execution, and is plotted using plotly.
