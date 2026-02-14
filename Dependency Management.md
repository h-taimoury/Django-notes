# Dependency Management

Here's the step by step process for dependency management in a Python project:

**1. Create and activate a virtual environment**

Before anything else, create a virtual environment for your project. This isolates your project's packages from the rest of your system so that only the packages your project actually needs end up being tracked.

```bash
python -m venv .venv
source .venv/bin/activate   # on Windows: .venv\Scripts\activate
```

**2. Install your project's dependencies**

With the virtual environment active, install whatever packages your project needs:

```bash
pip install django djangorestframework
```

**3. Generate the requirements.txt file**

In the Python world, dependencies are stored in a file named `requirements.txt`. It contains the names of all the libraries your project uses along with their exact versions, like this:

```
Django==5.0.1
djangorestframework==3.15.1
```

To generate this file automatically, you run:

```bash
pip freeze > requirements.txt
```

What this command does is ask pip to list all packages currently installed in your active environment, and then the `>` operator writes that output directly into `requirements.txt`. This is why having a virtual environment is critical — if you run this command without one, pip sees every package installed on your entire system and dumps all of them into the file, most of which have nothing to do with your project.

**4. Push your code to GitHub**

Add `requirements.txt` to your project and push it to GitHub. Just make sure your `.venv/` folder is in `.gitignore` so you don't push the virtual environment itself — only the requirements file is needed.

```bash
git add .
git commit -m "add requirements.txt"
git push
```

**5. Another person sets up the project**

Someone else clones your repository, creates their own virtual environment, activates it, and then runs:

```bash
pip install -r requirements.txt
```

The `-r` flag tells pip to read the file and install every package listed in it, with the exact same versions you specified. This guarantees they end up with the same environment you developed with, avoiding the classic "it works on my machine" problem.
