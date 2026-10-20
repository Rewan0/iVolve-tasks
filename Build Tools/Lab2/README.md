# Lab 2: Building and Packaging a Java Application with Gradle

## Objective

Build, test, package, and run a Java Calculator application using Gradle.

## Tools

- Ubuntu
- Java JDK 17
- Gradle 9.8.2
- Git

## Source Repository

[https://github.com/Ibrahim-Adel15/CalculatorGradleApp](https://github.com/Ibrahim-Adel15/CalculatorGradleApp)

## Execution Steps

### 1. Run Unit Tests

```bash
gradle test
```

Result: BUILD SUCCESSFUL.

### 2. Build and Package

```bash
gradle build
```

Result: BUILD SUCCESSFUL.

Generated artifact: `build/libs/calculator.jar`

### 3. Run the Application

```bash
java -jar build/libs/calculator.jar
```

### 4. Verify Results

Test inputs: 10 and 5.

- Sum: 15.0
- Subtract: 5.0
- Multiply: 50.0
- Divide: 2.0

## Screenshots

### 1. Test
<img width="1703" height="428" alt="image" src="https://github.com/user-attachments/assets/a7db54c1-723d-4d4c-bb6f-2e66b7f2079a" />


### 2. Build
<img width="1655" height="135" alt="GradleBuild" src="https://github.com/user-attachments/assets/b042dcea-49d7-4dfb-a242-5fad22802c1a" />

### 3. Run
<img width="1706" height="163" alt="image" src="https://github.com/user-attachments/assets/932f3d08-ff98-4ddc-9e9b-8fc665349552" />

## Conclusion

Gradle successfully built and packaged the Calculator application, and the unit tests passed. The generated JAR was verified as a build artifact.

