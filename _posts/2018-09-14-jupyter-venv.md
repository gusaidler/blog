---
layout: post
title: "How to create a Python virtual environment with Jupyter"
image: 
  path: /assets/images/posts/jupyter-venv/jupyter-post.png
  thumbnail: /assets/images/posts/jupyter-venv/jupyter-post.png
categories:
 - how to
tags:
 - python
 - jupyter
---
If you use Python, you probably create a separate virtual environment (venv) for each project, or at least you have heard that this a good practice to be followed. 

Here are some reasons for using it:
- It helps to maintain your system clean since you don’t install system-wide libraries that you are only going to need for a small project;
- It allows you to use a certain version of a library for one project and another version for another project;

So, basically, you have completely isolated Python environments for each of your projects, which can prevent big headaches (words from someone who had it already).

Until recently, I didn't know it is also possible to create a **venv** in a Jupyter environment, but it is, and it's quite easy!

## Here is how: 

#### 1. Navigate to the folder where your project will reside.
#### 2. Execute the following command to create a virtual environment:
	python3.6 -m venv venv

#### 3. Then activate it (so far exactly as "normal" Python):
	source venv/bin/activate

#### 4. Now, you have to install the `ipykernel` using `pip`:
	pip install ipykernel

#### 5. Once finished, you have to install a new `kernel`:
	ipython kernel install --user --name=new_project

#### 6. That's it! If you start Jupyter now, you will see that the `new_project` kernel is available to be used when creating new notebooks
![Jupyter]({{ '/assets/images/posts/jupyter-venv/jupyter-screen.png'|absolute_url}}){: .align-center}

**- All packages you install while having the `venv` activated  (steps 3 and 4) will be available exclusively for this new kernel**

These commands where executed on a Macbook with Python 3.6 installed, but it should be the same processe for every other system (you'd probably call `python` instead of `python3.6` though)
{: .notice--info}
