---
layout: post
title:  "Jupyter Notebook, Docker, Remote Servers"
author: Tung 
categories: [ Jekyll, tutorial, Docker ]
featured: true
image: assets/images/jupyter_docker.png
---

## What's the need to combine all these?
* When you just started with Machine Learning, Data Science, Deep Learning or just simply Kaggling, it is very tempting to install all the packages in one place. However, the nightmare starts when you have multiple projects with different CUDA toolkits package versions and dependencies. One does not simply have time to upgrade from CUDA 9.x to 10.x.
* There are also other times where you have the luxury to access into a very powerful remote server with the state-of-the-art GPU.You can ssh into the server and run your python script. But you also want to run jupyter notebook from your laptop and still tap into that precious GPU. What should you do now ?

<p align="center">
  <img src="{{ site.baseurl }}/assets/images/env_setup.png"/></p>
## Prerequisite
* You have a basic understanding what Docker, Jupyter Notebook and Remote Servers are
* A laptop of course ..


## Jupyter Notebook and Docker
First, let start with simpler things first. What we want is to run Jupyter Notebook together with a Docker Image from our local machine. You have to add the following line into your **~/.bash_profile**
```shell
kjupyter() {
    sudo docker run  --runtime=nvidia -v $PWD:/your/local/folder -v $PWD:/your/docker/folder-w=/your/docker/folder -p 8888:8888 --rm -it <yourdocker>:<tag> bash -c "export LD_LIBRARY_PATH=/usr/local/cuda/lib64; pip install jupyter_contrib_nbextensions; pip install jupyter_nbextensions_configurator; jupyter contrib nbextension install --user; jupyter nbextension install https://raw.githubusercontent.com/lambdalisue/jupyter-vim-binding/master/vim_binding.js --nbextensions=$(jupyter --data-dir)/nbextensions/vim_binding; jupyter notebook --notebook-dir=/tmp/working --ip='*' --port=8888 --no-browser --allow-root"
}
```
* This script above will bind **kjupyter** to running the docker , mounting your specified folder to your folder in the docker image, installing the relevant jupyter extensions (at least those are relvant to me, you can add in/remove those if your like)
* Some args that you need to pay attention to 
    * **--runtime=nvidia** : use this only if your docker utilize CPU
    * **/your/local/folder** and **/your/docker/folder** are the relevant folder you want to bind between your local machine and your docker image folder respectively

After that, you have to activate the **~/.bash_profile** by using 
```shell
source ~/.bash_profile
```

To run the docker, you just have to run the command above
```
kjupyter
```
## Jupyter Notebook and Remote Servers

Let's come to the second problem: To run jupyter notebook from your local machine that will be executed in a remote server. Firstly, lets create a new shell script called **jupyternotebook.sh**. The content of the shell script should be below

```shell
# $1 = username on remote

# $2 = remote IP

# $3 = jupyter port on remote

# $4 = jupyter port locally

ssh -N -f -L $4:localhost:$3 $1@$2

```

As explained in the script above, the first argument will be your username on the remote machine. The second argument will be the remote IP. The third and final argument will be the jupyter port on remote and local machine respectively <br/>

Lets run the script
```shell
bash jupyternotebook.sh dao 10.20.30.40 8181 8181

```

Next, you have to open anther terminal and  ssh into the remote server and run 

```shell
jupyter notebook --no-browser --port 8181
[I 18:17:17.637 NotebookApp] Serving notebooks from local directory: /home/dao
[I 18:17:17.637 NotebookApp] The Jupyter Notebook is running at:
[I 18:17:17.637 NotebookApp] http://localhost:8181/?token=abcd12345
[I 18:17:17.637 NotebookApp] Use Control-C to stop this server and shut down all kernels (twice to skip confirmation).
[C 18:17:17.638 NotebookApp]

    Copy/paste this URL into your browser when you connect for the first time,
    to login with a token:
        http://localhost:8181/?token=acbd12345
```
**Note** :
* Please make sure that you have jupyter notebook installed in the remote server
* The port number on remote machine should be the same in both these commands

As you can see from the output above, you have a token to log in. Now , all we have left to do is to open your browswer in your local machine, log into jupyter notebook and paste the token

```shell
localhost:<yourport>
```


