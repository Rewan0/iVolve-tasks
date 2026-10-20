# Lab 1: Building and Packaging a Java Application with Maven

## Objective
Build, test, package, and run a Java Calculator application using Apache Maven.

## Tools
- Ubuntu
- Java JDK 17
- Apache Maven
- Git

## Source Repository
https://github.com/Ibrahim-Adel15/CalculatorMavenApp

## Execution Steps

### 1. Run Unit Tests
```bash
mvn test
```
Result: 5 tests passed, 0 failures, 0 errors, and 0 skipped tests.

### 2. Build and Package
```bash
mvn package
```
Result: BUILD SUCCESS.

Generated artifact: `target/calculator.jar`

### 3. Run the Application
```bash
java -jar target/calculator.jar
```

### 4. Verify Results
Test inputs: 10 and 5.

- Sum: 15.0
- Subtract: 5.0
- Multiply: 50.0
- Divide: 2.0

## Screenshots
<img width="1698" height="473" alt="image" src="https://github.com/user-attachments/assets/bcd898b7-c499-47c2-9daf-f4afbd590173" />



See the `screenshots/` directory for evidence of the tests, build, and application execution.

## Conclusion
All five unit tests passed, Maven packaged the application successfully, and the generated JAR ran correctly.
