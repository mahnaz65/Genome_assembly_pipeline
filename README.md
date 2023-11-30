# Genome_assembly_pipeline
To learn the steps of genome assembly using Hifiasm  
Genome assembly workshop

# Raw data

I created a subsample from HiFi PacBio read that I have. With these subsamples, you should run the analyses on your local computer. So far, I do not have Hi-C data
related to this sample, so, we run the pipeline without the Hi-C data.
As you can see in the names of the samples, one of them is smaller. you can try which one that works on your local computer. Please download the files from the links below:

https://drive.google.com/file/d/16gE0bZWr3eby1jbJR1oITroP3scXxwfJ/view?usp=drive_link
https://drive.google.com/file/d/1GtP7SLnQEGgsxywnZqsOaTIDWN5QYsLs/view?usp=drive_link

### Installation

#### Anaconda
If you just install Ubuntu, perhaps you need to install Anaconda or Miniconda. I prefer Anaconda. If you installed Ubuntu, so, you should know that
Ubuntu is Debian-based, so, always you should install packages for Debian. Follow the link below to install Anaconda.
https://docs.anaconda.com/free/anaconda/install/linux/

Please read the page carefully, and do exactly what it wrote. Just copy the commands in the order, change the directory when needed, and type yes when needed. All
is explained on this page better than my explanation.

#### JAVA

This is better to be installed as some software is java-based. They might be compatible with a different version, but we installed the last version and let's see if
some software needs an older version, we can install that version afterward.
To install java follow commands below:  
```
  sudo apt-get install openjdk-18-jre
  sudo apt-get install openjdk-18-jdk openjdk-18-jre-headless
```


