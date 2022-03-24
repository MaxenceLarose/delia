# Medical data formatting and pre-processing module
This package is a medical data formatting and pre-processing module whose main objective is to build an HDF5 dataset containing patients' medical images as well as binary label maps obtained from the segmentation of these images (if available). The HDF5 dataset is then easier to use to perform tasks on the medical data, such as machine learning tasks.

Anyone who is willing to contribute is welcome to do so.

## Motivation

**Digital Imaging and Communications in Medicine** ([**DICOM**](https://www.dicomstandard.org/)) is *the* international standard for medical images and related information. It defines the formats for medical images that can be exchanged with the data and quality necessary for clinical use. The working group [DICOM WG-23](https://www.dicomstandard.org/activity/wgs/wg-23/) on Artificial Intelligence / Application Hosting is currently working to identify or develop the DICOM mechanisms to support AI workflows, concentrating on the clinical context. Moreover, their future *roadmap and objectives* includes working on the concern that current DICOM mechanisms might not be adequate to cover some use cases, particularly bulk analysis of large repository data, e.g. for training deep learning neural networks. However, no tool has been developed to achieve this goal at present.

The **purpose** of this module is therefore to provide the necessary tools to facilitate the use of medical images in an AI workflow.  This goal is accomplished by using the [HDF file format](https://www.hdfgroup.org/) to create a dataset containing patients' medical images as well as binary label maps obtained from the segmentation of these images (if available).

## Installation

### Latest stable version :

```
pip install dicom2hdf
```

### Latest (possibly unstable) version :

```
pip install git+https://github.com/MaxenceLarose/dicom2hdf
```

## Brief explanation of how it works 

First, it is necessary to explain briefly how the package builds a dataset using the provided data.

For each of the folders/patients contained in the `patients` folder, all DICOM files present in the `images` subfolder are read. If the series descriptions of a certain volume match one of the descriptions present in the given `series_descriptions` dictionary, this volume is automatically added to the patient dataset. Then, all DICOM-SEG files present in the `segmentations` folder are read. Segmentation volumes whose reference series UID matches one of the series UID read from the patient's `images` folder are added to the patient's dataset. The reference volume is also added to the patient dataset.

*Note : If no `series_descriptions` dictionary is given, the step of selecting the images according to this criterion is simply ignored. In the same way, if no DICOM-SEG file is present in the `segmentations` folder, the segmentations and images selection step according to this criterion is simply ignored.* 

I know, the above description is confusing. To fully understand it, read [Getting started](#getting-started).

## Getting started

### Organize your data

Since this module requires the use of data, it is important to properly configure the data-related elements before using it. The following sections show how to configure these.

#### File format

Images files must be in standard [**DICOM**](https://www.dicomstandard.org/) format and segmentations files must be in [DICOM-SEG](https://dicom.nema.org/medical/dicom/current/output/chtml/part03/sect_C.8.20.html) format.

If your segmentations files are in a research file format (`.nrrd`, `.nii`, etc.), you need to convert them into the standardized [DICOM-SEG](https://dicom.nema.org/medical/dicom/current/output/chtml/part03/sect_C.8.20.html) format. You can use the [pydicom-seg](https://pypi.org/project/pydicom-seg/) library to create the DICOM-SEG files OR use the [itkimage2dicomSEG](https://github.com/MaxenceLarose/itkimage2dicomSEG) python module, which provide a complete pipeline to perform this conversion.

#### Series descriptions (Optional)

*This dictionary is **not** mandatory for the code to work and therefore its default value is `None`.*

The series descriptions are specified as a **dictionary** that contains the series descriptions of the images that absolutely needs to be extracted from the patients' files. Keys are arbitrary names given to the images we want to add and values are lists of series descriptions. The images associated with these series descriptions do not need to have a corresponding segmentation volume. In fact, **the whole point of adding a way to specify which series descriptions should be added to the dataset is to be able to add images that have not been segmented.** Note that it can also be specified as a path to a **json file** that contains the series descriptions. Both methods are presented below.

**Warning!** *This dictionary (or json file) can be modified during the execution of the package functions. THIS IS NORMAL, we potentially want to add a new series description to this dictionary if none of the descriptions match the series in a patient's files.* 

##### Using a json file

Create a json file containing only the dictionary of the names given to the images we want to add (keys) and lists of series descriptions (values). Place this file in your data folder.

Here is an example of a json file configured as expected :

```json
{
    "TEP": [
        "TEP WB CORR (AC)",
        "TEP WB XL CORR (AC)"
    ],
    "CT": [
        "CT 2.5 WB",
        "AC CT 2.5 WB"
    ]
}
```

##### Using a Python dictionary

Create the organs dictionary in your main.py python file.

Here is an example of a python dictionary instanced as expected :

```python
series_descriptions = {
    "TEP": [
        "TEP WB CORR (AC)",
        "TEP WB XL CORR (AC)"
    ],
    "CT": [
        "CT 2.5 WB",
        "AC CT 2.5 WB"
    ]
}
```

#### Structure your patients directory

It is important to configure the directory structure correctly to ensure that the module interacts correctly with the data files. The repository, particularly the `patients` folder, must be structured as follows. *The names of the folders and files can differ, but they must be consistent with your own folders and files names*.

```
|_📂 Project directory/
  |_📄 main.py
  |_📂 data/
    |_📄 series_descriptions.json
    |_📂 patients/
      |_📂 patient1/
       	|_📂 images/
       	  |_📄 IM0.dcm
       	  |_📄 IM1.dcm
       	  |_📄 ...
       	|_📂 segmentations/
       	  |_📄 IM0.SEG.dcm
       	  |_📄 IM1.SEG.dcm
       	  |_📄 ...
      |_📂 patient2/
       	|_📂 images/
       	  |_📄 IM0.dcm
       	  |_📄 ...
       	|_📂 segmentations/
       	  |_📄 IM0.SEG.dcm
       	  |_📄 ...
      |_📂 ...
```

### Import the package

The easiest way to import the package is to use :

```python
from dicom2hdf import *
```

This will import the useful classes `PatientDataset` and `PatientDataGenerator` and the useful function `logs_file_setup`. These two classes represent two different ways of using the package. The following examples will present both procedures.

### Use the package

The two examples below show code to add to the `main.py` file. 

#### Example using the patient dataset class

This file can then be executed to obtain an hdf5 dataset.

```python
import logging

from dicom2hdf import logs_file_setup, PatientDataset

logs_file_setup(level=logging.INFO)

dataset = PatientDataset(
    path_to_dataset= "data/patient_dataset.h5",
)

dataset.create_hdf5_dataset(
    path_to_patients_folder="data/patients",
    images_folder_name="images",
    segmentations_folder_name="segmentations",
    series_descriptions="data/series_descriptions.json",
    verbose=True,
    overwrite_dataset=True
)

```

The created HDF5 dataset will then look something like :

![patient_dataset](https://github.com/MaxenceLarose/dicom2hdf/raw/main/images/patient_dataset.png)

#### Example using the patient data generator class

This file can then be executed to perform on-the-fly tasks on images.

```python
import logging

from dicom2hdf import logs_file_setup, PatientDataGenerator
import SimpleITK as sitk

logs_file_setup(level=logging.INFO)

patient_data_generator = PatientDataGenerator(
    path_to_patients_folder="data/patients",
    images_folder_name=¨images¨,
    segmentations_folder_name=¨segmentations¨,
    series_descriptions="data/series_descriptions.json",
    verbose=True
)

for patient_dataset in patient_data_generator:
    patient_name = patient_dataset.patient_name
    
    for patient_image_data in patient_dataset.data:
        dicom_header = patient_image_data.image.dicom_header
        simple_itk_image = patient_image_data.image.simple_itk_image
        numpy_array_image = sitk.GetArrayFromImage(simple_itk_image)
        
        """Perform any tasks on images on-the-fly like printing the image array shape."""
        print(numpy_array_image.shape)
```

### TODO

- [ ] Remove time sleep to handle tqdm console output.

- [ ] Set up logging to fit the expected configuration in a library.

- [ ] Use patients' unique identifiers rather than their names.
- [ ] Add a way to specify an opening mode ("w", "r", "a", etc.) when creating an instance of the `PatientDataset` class.
- [ ] Generalize the use of a specific tag to add images that have not been segmented. At the moment, the only tag available is `series_descriptions`.
- [ ] Add a parameter named `tags_to_used_as_attributes ` to the function `create_hdf5_dataset ` of the class `PatientDataset`. This parameter allows to choose the tags to be used as attributes for the images in the hdf5 file.

## License

This code is provided under the [Apache License 2.0](https://github.com/MaxenceLarose/dicom2hdf/blob/main/LICENSE).

## Citation

```
@article{dicom2hdf,
  title={dicom2hdf: DICOM to HDF python module},
  author={Maxence Larose},
  year={2022},
  publisher={Université Laval},
  url={https://github.com/MaxenceLarose/dicom2hdf},
}
```

## Contact

Maxence Larose, B. Ing., [maxence.larose.1@ulaval.ca](mailto:maxence.larose.1@ulaval.ca)
