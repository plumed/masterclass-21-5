#  PLUMED Masterclass 21.5: Simulations with multiple replicas

This lesson was given as part of the PLUMED masterclass series in 2021.  It includes:

* A video that explain the theory covered and a second video which shows you how the exercises should be completed.
* A series of exercises that you should try to complete yourself.
* Some supplementary python notebooks that provide further background information on the exercise.

The flow chart shown below indicates the order in which you should consult the resources.  You can click on the nodes to access the various resources.  Follow the thick black lines for the best results.  The resources that are connected by dashed lines are supplmentary resources that you may find useful when completing the exercise. 

This lesson was the fifth masterclass in the 2021 series.  You are strongly recommended to complete the masterclasses on umbrella sampling and metadynamics that are indicated in the flow chart below before tackling this masterclass. 

```mermaid
flowchart LR;
  A[Umbrella sampling] ==> C[Lecture I] 
  B[Metadynamics] --> C
  C ==> D[Instructions];
  D ==> E[Lecture II];
  D --> F[solution];
  click A "ref1" "This lesson introduces shows you how to run umbrella sampling calculations. During the exercises on replica exchange you rerun the umbrella sampling simulations from this earlier lesson with replica exchange. To get the most out of this exercise you should compare the result you obtain with the replica exchange here with the result you got from the earlier exercise without replica exchange";
  click B "ref2" "This lesson shows you how to run metadynamics simulations. In the exercise on running multiple replicas this method is used in tandem with replica exchange.";
  click C "video1" "A lecture that was given on March 15th 2021 as part of the plumed masterclass series that introduces you to the exercises in this lesson"
  click D "INSTRUCTIONS.md" "The instructions for the exercises";
  click E "video2" "A lecture that was given on March 22nd 2021 as part of the plumed masterclass series that goes through the solutions to the exercises in the lesson"
  click F "Solution.ipynb" "A python notebook that contains a full set of solutions to the exercises that are discussed in the masterclass.  This notebook is the one that was editted during the section video lecture of the masterclass"
```
