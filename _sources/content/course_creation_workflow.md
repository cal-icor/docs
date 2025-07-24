# Course Creation Workflow

Below is a typical set of steps for creating a course that uses JupyterHub:

1. **Create a private GitHub repository** for instructor notebooks that include solutions.
2. **Create a public GitHub repository** for student-facing notebooks.
3. **Author your notebooks.** The [Resources and Screen Recordings](#resources-and-screen-recordings) section below lists two excellent resources to help you get started.
4. **Copy the student versions** of the notebooks into the public repository.
5. **Choose a method for distributing notebooks to students:**
   - Use **nbgitpuller links** to allow students to load each notebook directly (see the resources section for guidance).
   - Have students **download notebooks from GitHub** and upload them to the Hub manually.
6. **Optional: Upload datasets to the `shared_readwrite` folder** of your institution's hub if you are providing datasets specific to your course. This allows students to access them directly from the `shared` folder.
   - The `shared` folder is **read-only** and accessible to all students and instructors on a given JupyterHub. This allows users to access datasets from the hub instead of each user uploading the same dataset to their home folder.
   - The `shared_readwrite` folder is **read and write** and accessible to instructors.
   - If you need access to the `shared_readwrite` folder, please [open an issue here](https://github.com/cal-icor/cal-icor-hubs/issues/new?template=admin_request.yaml).
   - More details about these folders can be found in the [Materials Sharing](notebooks_materials.md#materials-sharing) section.
7. **Decide how students will submit their work.** Most commonly, students download notebooks from the Hub and upload them to your LMS (e.g., Canvas) or a grading platform such as Gradescope.

---

# Resources and Screen Recordings

- [**DS Modules**](https://ds-modules.github.io/curriculum-guide/workflow/creating-notebooks.html) — Created by UC Berkeley’s Eric Van Dusen, this guide offers an excellent overview of notebook creation. While the documentation references `datahub.berkeley.edu`, you can apply the same process to your own hub at `<institution>.jupyter.cal-icor.org`.

- [**Authoring Notebooks: Start to Finish**](https://www.data8.org/zero-to-data-8/authoring/authoring_screen_recordings.html) — This resource includes five screen recordings covering:
  - GitHub repository creation
  - Authoring notebooks with and without `otter-grader`
  - Creating `nbgitpuller` links for distributing materials to students
