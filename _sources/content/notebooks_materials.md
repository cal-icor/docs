# Notebooks and Material Sharing

## Notebooks

We showcase Jupyter Notebook-based courses that highlight interactive and data-driven learning. Our goal is to build a collaborative community where educators contribute, share, and refine courses designed around Jupyter notebooks. By fostering an open and supportive environment, we aim to make high-quality, interactive learning materials more accessible and widely available.

Our current courses and documentation related to creating Jupyter notebooks are found [in our modules showcase](https://cal-icor.github.io/textbook/intro.html).  After selecting a module from the front page, you can then launch the notebook by clicking on the "Play" button in the top right corner of the module page and entering the URL of your JupyterHub deployment under "Private".

## Materials Sharing

For sharing datasets for coursework, we have shared folders available on your institution's JupyterHub deployment. These folders are designed to facilitate easy access to datasets for students and instructors alike.

Two folders are available:

- **`shared`**: This folder is **read only**, and accessible to all students and instructors on a given Jupyerhub. This is where students will access datasets that are commonly used across various courses instead of downloading them individually.
- **`shared_readwrite`**: This folder is **read and write**, and accessible to instructors only. Instructors can use this folder to upload datasets that are specific to their courses, allowing students to access them directly from the shared location.

As an instructor, you should have Administrator access before using these shared folders. If you are not an Administrator and need access to the `shared_readwrite` folder, please [open an issue here](https://github.com/cal-icor/cal-icor-hubs/issues/new?template=admin_request.yaml).

To best use these shared folders, we recommend the following steps:

1. **Create a subfolder in the `shared_readwrite` folder** for your course. This helps keep datasets organized and easily accessible.
2. **Upload datasets to your course subfolder** in the `shared_readwrite` folder. This allows students to access them directly from the `shared` folder without needing to download them individually.
3. **Include the `shared` folder path in any notebooks that you plan on sharing with the students.** This will allow you to test the notebooks with the correct paths to any datasets you have uploaded before you share them with students.

We encourage you to use these shared folders to streamline the process of providing datasets to your students, ensuring they have easy access to the materials they need for their coursework.
