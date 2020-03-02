SOFA Regression Documentation
==================

**This repository contains:**
- The tests to be perform on the scene for non regression: */Regression_test/*
- All the reference file of the tested scenes: */References*

** Building Regression_test program **
This repository should be included inside the main SOFA repository as /applications/projects/Regression folder.
Then while configuring SOFA cmake project, check the option: **APPLICAITON_REGRESSION_TEST** and be sure to also have **SOFA_BUILD_TESTS**.


# 1 - Non-Regression Test general mechanism #

## 1.a - Regression-tests files ##
The **Regression_test** program will search for **\*.regression-tests** files in the target directory (SOFA directory).
This file contains a list of scene to be tested. 
Each line of the list file must contain: 
- A local path to the scene
- The number of simulation steps to run
- A numerical epsilon for comparison 
- Optionally if mechanicalObject inside a mapped Node need to be tested.

See for example: SOFA_DIR/examples/RegressionStateScenes.regression-tests
```
### Demo scenes ###
Demos/caduceus.scn 100 1e-3
Demos/TriangleSurfaceCutting.scn 100 1e-4 1
Demos/liver.scn 100 1e-4 1
```

The class ```RegressionSceneList``` is used to parse folders and look for those files.
Right now we have:
- **For state tests:** RegressionStateScenes.regression-tests
- **For topology tests:** RegressionTopologyScenes.regression-tests


## 1.b - References
Reference files are stored in this repository under the **References/** folder, with the same folder hiearchy as the target scenes.
*For example, for Demo scenes inside SOFA, references will be stored inside **References/Demos/** *

**The reference files are generated when running the test for the first time on a scene and must be manually added to this repository. See section 3.a.**

If improvements of SOFA break some regression tests, the reference files of those tests need to be replaced by the new ground truth. 
To do that, the reference files must be locally deleted so they can be regenerated by running the tests. Then the new referenced files need to be pushed onto this repository to update the CI.


## 1.c - Non Regression Tests

For the moment non regression tests class are: 
- **StateRegression_test**: At each step, the state (position/velocity) of every independent dofs is compared to values in reference files.
- **TopologyRegression_test**: At each step, the number of topology element of every topology container is compared to values in reference files.


# 2 - How to run the regression tests

## 2.a - Full solution: By setting up environment
The program **Regression_test** can't take arguments but will use several Environment variables to run correctly:
- REGRESSION_SCENES_DIR : path to the root folder containing the scenes: SOFA_DIR
- REGRESSION_REFERENCES_DIR : path to the root folder containing the references
- SOFA_ROOT : path to sofa build directory.

The script **regression_test.py** will execute the program and add the arguments as Env variables. See example of use inside the script.

## 2.b - Quick solution: By using python script

# 3 - How to add new tests

## 3.a - Add a new scene to be tested

If the scene is already in Sofa repository: 
1. Just add its path inside the corresponding file: *Examples/RegressionStateScenes.regression-tests* for state test or *Examples/RegressionTopologyScenes.regression-tests* for topology test.
2. Create the reference file by runing the **Regression_test** program.
3. Push the list file into sofa-framework/sofa repository and the references into the sofa-framework/regression repository.

If the scene is inside a plugin:
Create first a new *.regression-tests* file inside your plugin and do the 3 steps above.

# 4 - Add a new non-regression test type

For each line of the *regression-tests* file, the class ```RegressionSceneList``` store all the parameters, as well as the scene path and its reference path, inside the a structure ```RegressionSceneData``` .

The result of this process is thus a vector of ```RegressionSceneData``` structures: ```std::vector<RegressionSceneData> m_scenes;```

Then, for each ```RegressionSceneData``` a gtest is created and the method ```runTest``` from the Regression_test class will be called to really perform the test (or create the reference).

To add a new type of test you will need to add:

- Inside the **RegressionSceneList** file:
  - Update the ```RegressionSceneData``` structure to store, per scene, the data needed for the test.

- Inside the **Regression_test** file:
  - Add a new **xxRegressionSceneList** to specify the targeted name file. For example ```static struct StateRegressionSceneList : public RegressionSceneList``` is looking for "RegressionStateScenes.regression-tests" files.
  - Add a new class **xxRegression_test** inheriting from **BaseRegression_test**. And implement the method ```void runTestImpl(RegressionSceneData data, sofa::simulation::Node::SPtr root, bool createReference = false);``` with the wanted test. 
  This method is called by ```BaseRegression_test::runTest``` which will check for the scene.
  
  See for example ```class StateRegression_test : public BaseRegression_test``` where runTestImpl check At each step, the state (position/velocity) of every independent dofs is compared to values in reference files and if the mean difference per node exceed threshold this will be reported as an error.

- Finally in Regression_test.cpp you need to add the code to create a gtest for each RegressionSceneData of your list with your xxRegression_test::runTest method.

See for example the code of StateRegression_test:
```
/// Create one instance of StateRegression_test per scene in stateRegressionSceneList.m_scenes list
/// Note: if N differents TEST_P(StateRegression_test, test_N) are created this will create M x N gtest. M being the number of values in the list.
INSTANTIATE_TEST_CASE_P(Regression_test,
    StateRegression_test,
    ::testing::ValuesIn( stateRegressionSceneList.m_scenes ),
    StateRegression_test::getTestName);

// Run state regression test on the listed scenes
TEST_P(StateRegression_test, sceneTest)
{
    runTest(GetParam());
}
```
