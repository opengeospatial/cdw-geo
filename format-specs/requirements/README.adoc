This folder contains requirements description.

Each file is a single requirement. The naming convention for these files is:

"REQn.adoc" where "n" corresponds to the requirement number. Numbers should have preceding zeros appropriate for the total number of requirements in the project (e.g., the first requirement could be REQ001 if less than 1000 requirements are anticipated).

The requirement files are integrated into the main document as links.

The requirement is expressed according to this pattern:

NOTE: for each requirement, there should be a corresponding Abstract Test in the "abstract_tests" folder.

NOTE: sample code may reference one or more requirements and should state which requirements are included in the code by adding the following line to the Extended Description:

"#REQS: reqnum1,reqnum2,...reqnumn"
