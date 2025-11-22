# Django Environment Setup

## We need Python installed on our machine

So visit python.org and download and install the latest version or just install Anaconda. By installing Anaconda, we get Python and other most used Python libraries.
After that check python being installed by

```
py --version
```

## Create a folder:

Create a folder for our backend. Let's name it 'backend' and cd into it.

## Create a virtual environment

Use venv module, which is a part of Python's standard library, to create a virtual environment

```
py -m venv .venv
```

This command will create a '.venv' folder inside of the 'backend' folder.

The main external package that provides similar functionality is virtualenv.

---

#### Note: Python Version Management and Virtual Environments on Windows

#### Question

It seems common to have multiple Python versions installed on a Windows machine. For instance, I might install Python directly from the official python.org website and later install a distribution like Anaconda, resulting in two separate Python installations.

My first question is: What are the other typical ways one might end up with different Python versions on a single Windows system?

My second question is: When using the standard command to create a virtual environment, `py -m venv .venv`, which specific Python interpreter's version will be used for this new environment?

---

#### Answer

You've highlighted an excellent point. Managing different Python versions is a key challenge in development, and Windows users frequently encounter this.

##### 1. Additional Ways to Acquire Multiple Python Versions

The standard installer and distributions like Anaconda are the most common culprits, but several other mechanisms can lead to multiple Python versions on your machine:

- **Version Management Tools:** Dedicated tools such as **`pyenv-win`** are specifically designed to streamline the installation and management of various Python versions (e.g., 3.8, 3.10, 3.12). These tools create distinct, isolated installations, making it simple to switch between versions per project.
- **Specialized Development Environments:** Certain IDEs (like PyCharm or VS Code) or development utility suites may prompt you to install a project-local or necessary Python interpreter, contributing to your overall count of installed versions.
- **Windows Subsystem for Linux (WSL):** If you utilize WSL, the Python installation within your Linux distribution (e.g., Ubuntu) is entirely separate and distinct from any Python versions installed natively on the Windows operating system.
- **Third-Party Application Dependencies:** Various applications or system tools may silently install their own embedded Python interpreters to function, which are usually not intended for general user development.
- **Containerization Tools:** While not a direct system installation, tools like Docker often involve setting up containers based on specific Python versions, further demonstrating the need for version isolation.

##### 2. Determining the Interpreter for `py -m venv .venv`

The crucial factor in determining the version of a new virtual environment is **which specific Python interpreter is executing the `venv` module.**

In your example, `py -m venv .venv` uses the Python interpreter that the **Python Launcher for Windows (`py.exe`)** resolves to based on its selection logic.

**Selection Logic for `py` (without a version flag):**

1.  **Explicit Version Check (Overridden by Flag):** If you had run a command like `py -3.11 -m venv .venv`, the Launcher would have explicitly chosen the latest installed **Python 3.11** interpreter to execute the command.
2.  **Default Interpreter:** Since you did not provide a version flag, the `py` launcher defaults to the following priority:
    - It typically selects the **highest installed Python 3 version** it finds (e.g., if both 3.9 and 3.12 are present, it selects 3.12).
    - This behavior can be modified by setting the environment variable `PY_PYTHON` to specify a preferred default version.

**Conclusion:**

The new virtual environment, `.venv`, will inherit the exact version (and its standard libraries) of the Python interpreter that was selected by the **Python Launcher's default mechanism**.

**Best Practice:**

To eliminate all ambiguity and ensure project fidelity, it is best practice to explicitly instruct the launcher which version to use:

| Command                  | Resulting Python Version in `.venv`                                              |
| :----------------------- | :------------------------------------------------------------------------------- |
| `py -3.11 -m venv .venv` | **Guaranteed** to use the latest installed Python **3.11**.                      |
| `py -m venv .venv`       | **Relies on the system default** (usually the newest 3.x found by the launcher). |

---

## Activate virtual environment

For a Bash terminal on windows run

```
source .venv/Scripts/activate
```

to deactivate the virtual environment, just run

```
deactivate
```

Everything we need for our project must be installed in the virtual environment. So when installing a package for the project, remember to activate the virtual environment first.

## Install Django

```
pip install Django
```

To check Django installation, just simply run

```
django --version
```

another way is to open Python REPL by running

```
py
```

Then run

```
import django
django.get_version()
```

To quit REPL, run

```
quit()
```

## Create a Django project

Many people use the name 'config' for the 'Project Package' and many people have struggled with the same issue you went through. The Django team made a mistake with the below command many years ago and while it's a very confusing folder structure, they can't get out of it.

Note: When we create a Django project, Django writes some comments in some modules. In one of these modules the comments say something like "Hello from 'Project_Package_Name' project". So Django team considers the name you provide for project package to be your project's name, but obviously this can cause a lot of confusion. So people avoid it and many use 'config' name for project package.

```
django-admin startproject config .
```

Here 'config' is the name of " Project Package" (containing a settings.py and other files). It's the container for all of your project's main settings and configurations.
It's a directory that is the actual Python package for your project. Its name is the Python package name you’ll need to use to import anything inside it (e.g. config.urls).

Let's always use the name "config" for Project Package until we find a better one because many developers do this.

But in many tutorials, they use confusing names for it like 'my_project' or 'my_django_project'. It's because Django team seems to prefer this convention of naming project package as your project's name. But it's confusing because it's not the project folder, it's project package folder. Even in Django documentation, a name of 'mysite' has been used which is not good.

The next parameter in above command is there to specify "Project Directory" (By default contains manage.py and a project package).

By using '.' (a period), I'm telling Django to setup the project here in 'backend' folder and next to '.venv' folder.

Remember to put the '.' at the end or specify another location for Django project stuff. If not, Django will create a directory named as 'config' in 'backend' folder for all the Django stuff, so the same name as project package. So we'll end up having two nested folders with same name which is very confusing.

## Start the server:

Run:

```
py manage.py runserver
```
