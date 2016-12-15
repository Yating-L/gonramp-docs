Introduction to G-OnRamp Walkthrough
====================================

Introduction
------------

The overall goal of this walkthrough is to illustrate the use of the G-OnRamp workflow to create a genome browser. The walkthrough will cover topics including how to create an account and login, how to import the G-OnRamp workflow into your workspace, how to modify the workflow by changing tool parameters and adding new tools, and how to run the workflow. This walkthrough assumes that the reader has some basic familiarity with Galaxy. (See the “Overview of Galaxy” presentation for a detailed introduction to Galaxy). 

Login
------------------

Start the G-OnRamp virtual image and start the Galaxy server. Click on the link `http://192.168.56.11:8080/gonramp/ <http://192.168.56.11:8080/gonramp/>`_ to navigate to the G-OnRamp instance.

First of all, you need to log into your Galaxy account in order to access the full functionality of Galaxy. First click on "User" at the menu bar, and then choose "Login" (:numref:`figure-login`). Then enter your username and password (:numref:`figure-login-form`).

    - username: galaxyadmin    
    - password: 12341234

.. _figure-login:

.. figure:: ./image/Login1.png
   :height: 200px
   
   Open the login form from the “User” menu at the main menu bar
   
.. _figure-login-form:

.. figure:: ./image/Login2.png
   :height: 200px
   
   Enter the email address that you used to create your Galaxy account and your account password to log into Galaxy
   
Import G-OnRamp workflow
------------------------

The G-OnRamp workflow has been shared with you in the "Shared Data". Click on "Shared Data" in the top menu bar and choose "Workflows" in the drop down menu (:numref:`share-workflows`). Find the workflow called "G-OnRamp" in the published workflows and click on the down arrow on the right side. Choose "import" to import the G-OnRamp workflow into your Galaxy account (:numref:`import-workflow`). 

.. _share-workflows:

.. figure:: ./image/workflow.png
   :height: 200px

   Find the G-OnRamp workflow at "Shared Data" in the top menu bar
   
.. _import-workflow:

.. figure:: ./image/import-workflow.png

   Click on the down arrow on the right side and import the workflow 

You can access the imported workflow via the “Workflow” menu item on the menu bar. The “imported: G-OnRamp” workflow will appear under the “Your workflows” section. When you right click (control-click on the Mac) on the workflow or click on the down arrow, a context-sensitive menu will appear where you can edit, run, share or download, copy, rename, view, and delete the workflow (:numref:`workflow`).

.. _workflow:

.. figure:: ./image/workflow2.png
   
   Context-sensitive menu that specifies the different ways that you can manipulate the imported workflow

Enter “G-OnRamp: D. biarmipes F element” into the “Workflow Name” field and then click on “Rename” to change the name of the workflow (:numref:`rename`). Go back to the home page.

.. _rename:

.. figure:: ./image/rename.png

   Rename the workflow

Run the G-OnRamp workflow
-------------------------

Upload your datasets
####################

The test datasets that we will use in this walkthrough are available at the "Data Libraries" within the “Shared Datasets” (:numref:`data-lib`). Click on the folder "Dbia3". There are four test datasets in the folder: the reference genome sequence from the Drosophila biarmipes Muller F element, a collection of Drosophila melanogaster protein sequences, and the forward and reverse paired-end reads from contig16 of D. biarmipes RNA-Seq data. Select all the datasets in the foder(:numref:`import-data`). Import them to a new History, which you can name it "Dbia3" (:numref:`import-data2`).

.. _data-lib:

.. figure:: ./image/data-lib.png
   
   Go to "Shared Data" and choose "Data Libraries" from the drop down menu

.. _import-data:

.. figure:: ./image/import-data.png
   
   Select all the datasets in the Dbia3 folder  

.. _import-data2:

.. figure:: ./image/import-data2.png
   :height: 250px
   
   Create a new History named Dbia3 and import the datasets to the new History  

Once all of your datasets have been import to History, you can use the G-OnRamp workflow to analyze your datasets. However, before running the G-OnRamp workflow, you need to verify that Galaxy has assigned the correct data format to each of the file. In some cases, Galaxy incorrectly assigned the “dmel-all-translation-r6.11.fasta” file to the sdf format instead of the fasta format. Therefore, you need to manually edit its datatype by clicking on “pencil” icon at the top right corner of this dataset (:numref:`datatype`). You can skip this datatype editing by going directly to :ref:`section-workflow` if all your files have been assigned the correct data format. 

.. _datatype:

.. figure:: ./image/datatype.png
   :height: 200px
   
   Edit the attributes of the dataset by clicking on the "pencil" button

Click on the Datatype tab, click on the down arrow under the “New Type” field, and then select “fasta” (:numref:`datatype2`). Don’t forget to click on the “Save” button (:numref:`datatype3`).

.. _datatype2:

.. figure:: ./image/datatype2.png
   
   Change the datatype of the dmel-all-translation-r6.11.fasta dataset to fasta
   
.. _datatype3:

.. figure:: ./image/datatype3.png
   :height: 200px
   
   Save the new datatype for the dmel-all-translation-r6.11.fasta dataset

.. _section-workflow:

Run the workflow
################

Click on the “Workflow” menu on the menu bar. Click on the “G-OnRamp:D. biarmipes F element” workflow and then select “Run” from the drop-down menu to run the workflow (Figure 15).

.. _run-workflow:

.. figure:: ./image/run-workflow.png
   :height: 300px
   
   Click on the drop-down menu for the “G-OnRamp: D. biarmipes F element” workflow to run the workflow

The workflow consists of 16 different steps. In order to run the workflow, you need to specify the datasets that should be used in Step 1 to Step 4 (:numref:`specify-data`). Select the reference genome for Step 1. Select the protein sequence(s) that you would like to align against the assembly in Step 2. Select the forward and reverse reads from your RNA-Seq experiments for Step 3 and Step 4. 

.. _specify-data:

.. figure:: ./image/specify-data.png
   
   Specify the datasets that should be used in Step 1 to Step 4 

You can also use this form to examine and edit the parameter settings for each step of the workflow. Let’s run the workflow with the default settings this time. We will cover how to modify the G-OnRamp workflow in a subsequent walkthrough. (See the “Customize the Genome Browsers produced by G-OnRamp” walkthrough for details.)

Finally, scroll down to the bottom of the page. Check the box to send the results to a new History and then click on the “Run workflow” button to start the analysis (:numref:`run-workflow2`) .

.. _run-workflow2:

.. figure:: ./image/run-workflow2.png
   :height: 200px
    
   Click on the "Run workflow" button to start the analysis

View the results
################

After all the steps in the G-OnRamp workflow are complete (which will take a few minutes), you can view the genome browser for the D. biarmipes F element by expanding the “Hub Archive Creator” step (i.e. Step 18) and then click on the “main” link next to the “display at Track Hub UCSC” field (:numref:`main`). 

.. _main:

.. figure:: ./image/main.png
   :height: 200px
   
   View results on the UCSC genome browser

Because the RNA-Seq dataset that we used in this analysis only contains the RNA-Seq reads that mapped to contig16, we will examine this contig using the UCSC Genome Browser. Enter “contig16” into the “Position/Search Term” field and then click on the GO button (:numref:`contig16`). Finally, you can then see the results of the G-OnRamp workflow as different evidence tracks on the UCSC genome browser (:numref:`browser`).

.. _contig16:

.. figure:: ./image/contig16.png
   :height: 200px
   
   Specify the contig number in the “Position/Search Term” field and then click on GO
   
.. _browser:

.. figure:: ./image/browser.png
   :height: 200px
   
   View the genome assembly and the evidence tracks produced by the Hub Archive Creator on the UCSC genome browser
   


   