# Using SSH Key for Git Repo

## Prerequisite

### On Windows

1. ***Git bash*** : We use [Git for Windows](https://gitforwindows.org/) here.

2. `ssh-keygen`: Usually included in ***Git bash***.

### On Mac

1. `Terminal.App` : By default, you already have it.

2. `ssh-keygen` : Usually you already have it.


## Generating a Key

### Command
User terminal or git bash and type ***like*** below (Don't just copy it!)

```
ssh-keygen -t ed25519 -C "YourEmail@Domain.com" -N PassPhraseToUseTheKey -f path/to/store/KeyFileName
```


### Explanations

1. `ssh-keygen`: The application name you need to generate ssh key.

2. `-t` : Specify which type of public key algorithm to use for the key generation.   
For more other options, please refer to the manual (`man ssh-ketgen`).

3. `-C` : Comment to the key.

4. `-N` : A pass phrase to access the key, without which you can't use it.

5. `-f` : Specify the location and file name(s) to store the generated key content.    
Two files will be generated, `KeyFileName.pub` and `KeyFileName` under the director `path/to/store`.   
And `KeyFileName` is the private key which you should never tell other, and `KeyFileName.pub` to deploy with the git repository.


## Deploying the Key

Maybe not going to write about this.


## Cloning Repo with the Key

### Command
User terminal or git bash and type ***like*** below (Don't just copy it!)

```
ssh-agent bash -c 'ssh-add path/to/store/KeyFileName; git clone git@github.com:YourName/YourRepo.git'
```

### Explanations

1. `ssh-agent`: The application you need for executing the command.

2. `bash`: The shell to use to execute the command,

3. `-c`: What command to execute.

4. `ssh-add`: Add an ssh key to the shell execution context


## Using the Key

Without proper configuration, accessing to a remote repository located by SSH Uri will see error message like below:

```
Permission denied (publickey).
fatal: Could not read from remote repository.
```

That's because, we clone the repository with SSH key though,    
we haven't told git yet how to interact with remote repository with the key.


### Configuring the Repository

Make sure you've entered the repo directory first.

```
cd path/to/the/repo
```

And then we configure the ssh key ***ONLY FOR OUR REPO***.   

On *nix:

```
git config --local core.sshCommand "ssh -i path/to/PrivateKey -F /dev/null"
```

On Windows:

```
git config --local core.sshCommand "ssh -i path/to/PrivateKey"
```

To find out if the configuration works, you can try with `git fetch`.    
You are supposed to see pass phrase prompt like below:

```
Enter passphrase for key 'path/to/PrivateKey': 
```




