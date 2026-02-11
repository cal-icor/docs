# What is the Hub?

## Cal-ICOR: California Education Learning Lab and UC Berkeley

[California Education Learning Lab](https://calearninglab.org/) and UC Berkeley are teaming up to provide browser-based computing environments for Data and Computer Science courses in California Higher Education.

**Website:** [https://cal-icor.org/](https://cal-icor.org/)

---

## The Big Picture

The JupyterHub deployments are addressing various challenges in teaching Data and Computer Science:

- **Consistent versions and environments** ensure every student works with the same package/library versions.
- **Consistent processing power and memory** eliminate dependency on students’ personal computer configurations.
- **No configuration required by students**, allowing them to access materials and start working immediately.
- **Authentication via your institution's credentials**, avoiding the need for separate accounts.
- **Streamlined material distribution** for easy and reliable access.
- **Persistent student storage**, removing the need for constant downloads of work.

---

## JupyterHub Configuration

The JupyterHub is tied to your institution:  
`<institution>.jupyter.cal-icor.org`

The following is provided:

- **Custom Environment (Docker image):** You can maintain your own image or use ours. This image defines the compute environment students use.
- **Computing Resources:** By default, each user is allocated 1 CPU and 2 GB of memory. These limits can be customized as needed.
- **Development Tools:**
  - JupyterLab (with Python and R kernels)
  - VSCode
  - RStudio
  - Shiny
  - Terminal access, enabling students to run scripts outside of an IDE

- **Language Support:**
  - Python
  - R
  - C++

- **User Experience:**
  - Each user gets the same Python and R packages pre-installed
  - Shared directories (read/write for admins/instructors, read-only for everyone else)
  - *Important!* All user sessions have a hard limit of 12 hours.  If a user session is automatically killed at 12 hours, the user will need to re-authenticate to pick up where they were last at!