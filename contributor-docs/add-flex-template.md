# Adding a Flex Template

## Overview

Develop a simple Flex Template and run it on Google Cloud. This will take you
through setting up the POM and template code, staging it in Google Container
Registry, and running it on Dataflow. Once done, you'll be able to run the
template to calculate the frequency of each word in a given file in
Google Cloud Storage and output the results to another file in
Google Cloud Storage.

## Google Cloud Resources

1. Dataflow
2. Google Container Registry (GCR)
3. Google Cloud Storage (GCS)

## Create a Word Count Template

### Step 1: Create the module

Create a directory under `v2/` named `wordcount/` and add a `pom.xml` file with
the following content:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns="http://maven.apache.org/POM/4.0.0"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">

  <parent>
    <groupId>com.google.cloud.teleport.v2</groupId>
    <artifactId>dynamic-templates</artifactId>
    <version>1.0-SNAPSHOT</version>
  </parent>

  <modelVersion>4.0.0</modelVersion>
  <artifactId>wordcount</artifactId>

  <dependencies>
    <!-- Not always necessary, but sometimes Maven resolves to a Guava version
      that is incompatible with Google Cloud Storage, leading to a
      NoSuchMethodError. This forces a valid version to be used. -->
    <dependency>
      <groupId>com.google.guava</groupId>
      <artifactId>guava</artifactId>
      <version>${guava.version}</version>
    </dependency>

    <dependency>
      <groupId>com.google.truth</groupId>
      <artifactId>truth</artifactId>
      <version>${truth.version}</version>
      <scope>test</scope>
    </dependency>
  </dependencies>

</project>
```

This sets up `v2/wordcount` as a Maven module with a test-only dependency that
will be discussed later.

### Step 2: Add module to parent

In the POM, we have declared a parent named `dynamic-templates`. This
corresponds to the `pom.xml` file under `v2/`. Open that file, and towards the
bottom,
you should see a list of child modules. Add the `wordcount` module to this list:

```xml

<module>wordcount</module>
```

WARNING: Adding dependencies to `v2/pom.xml` can increase build time and
container sizes for *all* Flex Templates. If possible, please avoid inheriting
dependencies and only put the most general dependencies, like Beam and JUnit,
in the parent.

### Step 3: Add packages

Under `v2/wordcount/src/main/java`, add two packages named
`com.google.cloud.teleport.v2.templates` and
`com.google.cloud.teleport.v2.transforms`. Under each, create a
`package-info.java` file with the contents for the relevant package:

```java
/** Package for the template. */
package com.google.cloud.teleport.v2.templates;
```

```java
/** Package for the transforms. */
package com.google.cloud.teleport.v2.transforms;
```

### Step 4: Add the transforms

Create a file under the `transforms/` directory named `WordCountTransforms` and
add the following content:

```java
package com.google.cloud.teleport.v2.transforms;

import java.util.Arrays;
import org.apache.beam.sdk.metrics.Counter;
import org.apache.beam.sdk.metrics.Metrics;
import org.apache.beam.sdk.transforms.Count;
import org.apache.beam.sdk.transforms.DoFn;
import org.apache.beam.sdk.transforms.PTransform;
import org.apache.beam.sdk.transforms.ParDo;
import org.apache.beam.sdk.values.KV;
import org.apache.beam.sdk.values.PCollection;

/** A {@link WordCountTransforms} converts lines into tokens and counts words. */
public class WordCountTransforms {

  /**
   * A PTransform that converts a PCollection containing lines of text into a PCollection of word
   * counts.
   */
  public static class CountWords
      extends PTransform<PCollection<String>, PCollection<KV<String, Long>>> {

    @Override
    public PCollection<KV<String, Long>> expand(PCollection<String> lines) {

      // Convert lines of text into individual words.
      PCollection<String> words = lines.apply(ParDo.of(new ExtractWordsFn()));

      // Count the number of times each word occurs.
      return words.apply(Count.<String>perElement());
    }
  }

  static class ExtractWordsFn extends DoFn<String, String> {

    private final Counter emptyLines = Metrics.counter(ExtractWordsFn.class,
        "emptyLines");

    @ProcessElement
    public void processElement(@Element String line,
        OutputReceiver<String> receiver) {
      line = line.trim();
      if (line.isEmpty()) {
        emptyLines.inc();
      } else {
        // Split the line into words.
        String[] words = line.split("[^a-zA-Z']+");

        // Output each word encountered into the output PCollection.
        Arrays.stream(words).filter((word) -> !word.isEmpty())
            .forEach(receiver::output);
      }
    }
  }
}
```

This exposes a
[PTransform](https://beam.apache.org/documentation/programming-guide/#transforms)
for the template to eventually use. This transform utilizes a custom
[ParDo](https://beam.apache.org/documentation/transforms/java/elementwise/pardo/)
to handle the transformation. We could use the `ParDo` directly in the template
code, but `PTransform`s provide an easier-to-use and cleaner interface and make
it easier to test a full transform.

You may also notice that within the `DoFn`, we do not add empty lines to the
`receiver`, instead incrementing a counter. These lines serve no purpose for the
rest of the pipeline, but silently dropping them may also cause confusion.
Logging is another option, but with enough data fulfilling the logging
requirement, that can quickly lead to log spam, so using metrics, like a
counter, is preferable in this case.

### Step 5: Add the template code

Under the `templates` package, add a file named `WordCount.java` with the
following code and metadata annotations:

```java
package com.google.cloud.teleport.v2.templates;

import static com.google.common.base.Preconditions.checkNotNull;

import com.google.cloud.teleport.metadata.Template;
import com.google.cloud.teleport.metadata.TemplateCategory;
import com.google.cloud.teleport.metadata.TemplateParameter;
import com.google.cloud.teleport.v2.templates.WordCount.WordCountOptions;
import com.google.cloud.teleport.v2.transforms.WordCountTransforms;
import org.apache.beam.sdk.Pipeline;
import org.apache.beam.sdk.PipelineResult;
import org.apache.beam.sdk.io.TextIO;
import org.apache.beam.sdk.options.Default;
import org.apache.beam.sdk.options.Description;
import org.apache.beam.sdk.options.PipelineOptions;
import org.apache.beam.sdk.options.PipelineOptionsFactory;
import org.apache.beam.sdk.options.Validation.Required;
import org.apache.beam.sdk.transforms.MapElements;
import org.apache.beam.sdk.transforms.SimpleFunction;
import org.apache.beam.sdk.values.KV;
import org.apache.beam.sdk.values.PCollection;

/** Word count template. */
@Template(
    name = "Word_Count_Flex",
    category = TemplateCategory.GET_STARTED,
    displayName = "Word Count",
    description =
        "Batch pipeline to read text files from Cloud Storage and perform "
            + "frequency count on each of the words.",
    flexContainerName = "wordcount",
    optionsClass = WordCountOptions.class)
public final class WordCount {

  /**
   * The {@link WordCountOptions} class provides the custom execution options passed by the executor
   * at the command-line.
   */
  public interface WordCountOptions extends PipelineOptions {

    @TemplateParameter.GcsReadFile(
        order = 1,
        description = "Input file(s) in Cloud Storage",
        helpText =
            "The input file pattern Dataflow reads from. Use the example file "
                + "(gs://dataflow-samples/shakespeare/kinglear.txt) or enter the path to your own "
                + "using the same format: gs://your-bucket/your-file.txt")
    String getInputFile();

    void setInputFile(String value);

    @TemplateParameter.GcsWriteFolder(
        order = 2,
        description = "Output Cloud Storage file prefix",
        helpText = "Path and filename prefix for writing output files. Ex: gs://your-bucket/counts")
    String getOutputPath();

    void setOutputPath(String value);

    @TemplateParameter.Integer(
        order = 3,
        optional = true,
        description = "Maximum output shards",
        helpText =
            "The maximum number of output shards produced when writing. A higher number of shards"
                + " means higher throughput for writing to Cloud Storage, but potentially higher"
                + " data aggregation cost across shards when processing output Cloud Storage"
                + " files. Default is runner dependent.")
    @Default.Integer(-1)
    int getNumShards();

    void setNumShards(int value);
  }

  /**
   * The main entry-point for pipeline execution.
   *
   * @param args command-line args passed by the executor.
   */
  public static void main(String[] args) {
    WordCountOptions options =
        PipelineOptionsFactory.fromArgs(args).withValidation()
            .as(WordCountOptions.class);

    run(options);
  }

  /**
   * Runs the pipeline to completion with the specified options. This method does not wait until the
   * pipeline is finished before returning. Invoke {@code result.waitUntilFinish()} on the result
   * object to block until the pipeline is finished running if blocking programmatic execution is
   * required.
   *
   * @param options the execution options.
   * @return the pipeline result.
   */
  public static PipelineResult run(WordCountOptions options) {
    checkNotNull(options, "options argument to run method cannot be null.");
    Pipeline pipeline = Pipeline.create(options);

    PCollection<String> inputLines =
        pipeline.apply("ReadLines", TextIO.read().from(options.getInputFile()));

    PCollection<String> wordsCount = applyTransforms(inputLines);

    TextIO.Write writer = TextIO.write().to(options.getOutputPath());
    if (options.getNumShards() > 0) {
      writer = writer.withNumShards(options.getNumShards());
    }

    wordsCount.apply("WriteCounts", writer);

    return pipeline.run();
  }

  /**
   * Applies set of transforms on the given input to derive the expected output.
   *
   * @param lines Collection of text lines
   * @return the count of words with each line representing word and count in the form word: count.
   */
  public static PCollection<String> applyTransforms(PCollection<String> lines) {
    return lines
        .apply(new WordCountTransforms.CountWords())
        .apply(MapElements.via(new FormatAsTextFn()));
  }

  /** A SimpleFunction that converts a Word and Count into a printable string. */
  private static class FormatAsTextFn extends
      SimpleFunction<KV<String, Long>, String> {

    @Override
    public String apply(KV<String, Long> input) {
      return input.getKey() + ": " + input.getValue();
    }
  }
}
```

This is the actual template where we construct the pipeline's graph. Some
options are provided for getting input and output locations, along with
configuring the output.

In this template, we wrap a couple of `PTransforms` in a separate method named
`applyTransforms`. We could do these `apply` steps directly in the `run` method,
but this makes it easier to unit test.

`FormatAsTextFn` is an implementation of Beam's `SimpleFunction` and used as the
mapping method for
[MapElements](https://beam.apache.org/documentation/transforms/java/elementwise/mapelements/)
, which, along with the previously mentioned `ParDo`, is a core building block
of Beam pipelines.

### Step 6: Verify Pipeline Build

Go to the `DataflowTemplates/` directory (the parent of the `v2/` directory) and
run the following command:

```shell
mvn spotless:apply -pl v2/wordcount
```

This will format the code. If you try to build and get checkstyle violations,
this can solve many of them, though some will need to be addressed manually,
such as missing JavaDocs.

Once formatted, you can run (from the project's root as well):

```shell
mvn clean install -pl v2/wordcount -am -Dmaven.test.skip
```

The `-am` option guarantees that all the necessary local dependencies are
included in the build. You can ignore the error related to `v2/wordcount` if any error occurs.

`-pl v2/wordcount` is how we specify the target module, allowing us to only
build what we need. You can see all the available modules in the
`pom.xml` file.

Lastly, we use `-Dmaven.test.skip` to avoid running any tests, which we are going
to cover next.

### Step 7: Add a unit test

If using IntelliJ, open `WordCount.java` and hit `Ctrl + Shift + T` (`Cmd +
Shift + T` on Mac) to create a test file. Otherwise, create the
`com.google.cloud.teleport.v2.templates` package under `src/test/java` and add
`WordCountTest.java`. The contents of the test file should be the following:

```java
package com.google.cloud.teleport.v2.templates;

import static com.google.common.truth.Truth.assertThat;

import java.io.File;
import java.io.FileWriter;
import java.io.IOException;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import org.apache.beam.sdk.io.TextIO;
import org.apache.beam.sdk.testing.PAssert;
import org.apache.beam.sdk.testing.TestPipeline;
import org.apache.beam.sdk.values.PCollection;
import org.junit.ClassRule;
import org.junit.Rule;
import org.junit.Test;
import org.junit.rules.ExpectedException;
import org.junit.rules.TemporaryFolder;
import org.junit.runner.RunWith;
import org.junit.runners.JUnit4;

/** Test cases for the {@link WordCount} class. */
@RunWith(JUnit4.class)
public final class WordCountTest {

  @Rule
  public final transient TestPipeline pipeline = TestPipeline.create();

  @Rule
  public final ExpectedException expectedException = ExpectedException.none();

  @ClassRule
  public static TemporaryFolder tempFolder = new TemporaryFolder();

  @Test
  public void testWordCount_returnsValidCount() throws IOException {
    // Arrange
    String filePath = tempFolder.newFile().getAbsolutePath();
    writeToFile(filePath, Arrays.asList("Beam Pipeline", "Beam Java Sdk"));
    PCollection<String> inputLines = pipeline.apply("Read Lines",
        TextIO.read().from(filePath));

    // Act
    PCollection<String> results = WordCount.applyTransforms(inputLines);

    // Assert
    PAssert.that(results)
        .satisfies(
            pcollection -> {
              List<String> result = new ArrayList<>();
              pcollection.iterator().forEachRemaining(result::add);

              String[] expected = {"Beam: 2", "Java: 1", "Pipeline: 1",
                  "Sdk: 1"};

              assertThat(result.size()).isEqualTo(4);
              assertThat(result).containsExactlyElementsIn(expected);

              return null;
            });

    pipeline.run();
  }

  private void writeToFile(String filePath, List<String> lines)
      throws IOException {
    String newlineCharacter = "\n";
    try (FileWriter fileWriter = new FileWriter(new File(filePath))) {
      for (String line : lines) {
        fileWriter.write(line + newlineCharacter);
      }
    }
  }
}
```

All unit tests should follow the basic Arrange, Act, Assert structure, where
test data is prepared in Arrange, acted on in Act, and verified in Assert. The
block comments are unnecessary unless the block has multiple lines.

For verifying data, we encourage the use of
[Google Truth](https://github.com/google/truth), though you may see other
assertion libraries used in older templates. Please avoid using these. The only
exception is using `assertThrows` from JUnit, which does not have a good Truth
equivalent.

### Step 8: Run unit test

You can run the unit test with the following command:

```shell
mvn clean test -pl v2/wordcount -am \
  -Dtest=WordCountTest -DfailIfNoTests=false
```

This is similar to the above but with the `test` target specified. Since we will
be building other modules as well, we need to set the
[failIfNoTests](https://maven.apache.org/surefire/maven-surefire-plugin/test-mojo.html#failIfNoTests)
property as false to avoid failures in dependencies when no tests are run.

### Step 9: Stage or Run the template

`Stage` is the term used for the process of building and uploading the template to
Cloud Registry and Google Cloud Storage. After the template code is created, you
can use the 
[Templates Plugin](https://github.com/GoogleCloudPlatform/DataflowTemplates#templates-plugin)
to help with the staging process and/or run the template.

First of all, be sure to authenticate through `gcloud` and install the plugin:

```shell
gcloud auth login
gcloud auth configure-docker

# Install the plugin, used to stage and run the templates
mvn clean install -pl plugins/templates-maven-plugin -am 

# Configure the environment variables
export USERNAME=`whoami`
export PROJECT=<Your GCP projectid>
export REGION=us-central1
export BUCKET_NAME=<Your GCS Bucket name>
export MODULE_NAME=wordcount
```

Then choose between one of the options:

1. Just stage the template:
    ```shell
   mvn clean package -PtemplatesStage  \
    -DskipTests \
    -DprojectId="$PROJECT" \
    -DbucketName="$BUCKET_NAME" \
    -DstagePrefix="$USERNAME/$MODULE_NAME" \
    -DtemplateName="Word_Count_Flex" \
    -pl v2/$MODULE_NAME -am
    ```

2. Stage + run the template:
    ```shell
    mvn clean package -PtemplatesRun \
      -DskipTests \
      -DprojectId="$PROJECT" \
      -DbucketName="$BUCKET_NAME" \
      -Dregion="$REGION" \
      -DjobName="wordcount-$(date +'%Y%m%d%H%M%S')" \
      -DtemplateName="Word_Count_Flex" \
      -Dparameters="inputFile=gs://dataflow-samples/shakespeare/kinglear.txt,outputPath=gs://$BUCKET_NAME/output/wordcount/$USERNAME/wordcount" \
      -pl v2/$MODULE -am
    ```

Both commands should print what is the template location on Cloud Storage:

```
Flex Template was staged! gs://{BUCKET}/{PATH}
```

You can use that path to share the template. To run the template at any time
using `gcloud`, you can use:

```
export TEMPLATE_SPEC_GCSPATH={PATH_FROM_ABOVE}

gcloud dataflow flex-template run "wordcount-$(date +'%Y%m%d%H%M%S')" \
  --project "$PROJECT" \
  --region "$REGION" \
  --template-file-gcs-location "$TEMPLATE_SPEC_GCSPATH" \
  --parameters inputFile="gs://dataflow-samples/shakespeare/kinglear.txt"  \
  --parameters outputPath="gs://$BUCKET_NAME/output/wordcount/$USERNAME/wordcount"
```

Once ran, you can verify that the job ran successfully by going to the Dataflow
jobs page in the Google Cloud Console.

NOTE: If preferred, you can also launch the template from the Google
Cloud Console by selecting the `Custom Template` option in the dropdown of the
`Create job from template` page. You would then point it to the path specified
by `TEMPLATE_SPEC_GCSPATH`.

### Step 10: Cleanup

It's a good idea to run `mvn spotless:apply` after development and before
putting in a PR. This will fix any formatting issues.

You can delete all the code from this tutorial by running:

```shell
git stash save --include-untracked && git stash drop
```

If you created a separate project for this tutorial, also remember to delete it
to avoid any billing costs from the artifacts created by the template.

### Bonus Step: Celebrate!

You're done with this tutorial!
