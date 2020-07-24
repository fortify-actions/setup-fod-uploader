# Setup Fortify on Demand Uploader

Build secure software fast with [Fortify](https://www.microfocus.com/en-us/solutions/application-security). Fortify offers end-to-end application security solutions with the flexibility of testing on-premises and on-demand to scale and cover the entire software development lifecycle.  With Fortify, find security issues early and fix at the speed of DevOps. This GitHub Action sets up the Fortify on Demand (FoD) Uploader to integrate Static Application Security Testing (SAST) into your GitHub workflows. This action:
* Downloads and caches the specified version of the Fortify on Demand Uploader JAR file
* Adds the `FOD_UPLOAD_JAR` environment variable containing the full path to the Fortify on Demand Uploader JAR file

## Usage

FoD Uploader requires source code and dependencies to be packaged into a zip file. We recommend using the Fortify ScanCentral Client to perform the packaging before invoking FoD Uploader, as illustrated in the following example workflow:

```yaml
name: FoD SAST scan                                     # Name of this workflow
on:
  push:                                                 # Perform Fortify SAST on push and/or pull requests
    branches:
      - master
  pull_request:
    branches:
      - master
jobs:                                                  
  build:
    runs-on: ubuntu-latest                              # Use the appropriate runner for building your source code

    steps:
      - uses: actions/checkout@v2                       # Check out source code
      - uses: actions/setup-java@v1                     # Set up Java 1.8; required by ScanCentral Client and FoD Uploader
        with:
          java-version: 1.8

      ### Set up ScanCentral Client and FoD Uploader ###
      - uses: fortify/gha-setup-scancentral-client@v1   # Set up ScanCentral Client and add to system path
      - uses: fortify/gha-setup-fod-uploader@v1         # Set up FoD Uploader, set FOD_UPLOAD_JAR variable

      ### Package source code using ScanCentral Client ###
      - run: scancentral package -bt mvn -o package.zip

      ### Start Fortify on Demand SAST scan ###
      - run: java -jar $FOD_UPLOAD_JAR -z package.zip -aurl https://api.ams.fortify.com/ -purl https://ams.fortify.com/ -rid "$FOD_RELEASE_ID" -tc "$FOD_TENANT" -uc "$FOD_USER" "$FOD_PAT" -ep 2 -pp 1
        env:                                            
          FOD_TENANT: ${{ secrets.FOD_TENANT }}
          FOD_USER: ${{ secrets.FOD_USER }}
          FOD_PAT: ${{ secrets.FOD_PAT }}
          FOD_RELEASE_ID: ${{ secrets.FOD_RELEASE_ID }} 

      ### Archive ScanCentral logs and package ###
      - uses: actions/upload-artifact@v2                # Archive ScanCentral logs for debugging purposes
        if: always()
        with:
          name: scancentral-logs
          path: ~/.fortify/scancentral/log
      - uses: actions/upload-artifact@v2                # Archive ScanCentral package for debugging purposes
        if: always()
        with:
          name: package
          path: package.zip
```

This example workflow demonstrates the use of the `fortify/gha-setup-scancentral-client` and `fortify/gha-setup-fod-uploader` actions to set up ScanCentral Client and FoD Uploader respectively, 
and then invoking these utilities similar to how you would manually run these commands from a command line. All potentially sensitive data should be stored in the GitHub secrets storage.

Please see the following resources for more information:

* [FoD Uploader documentation](https://github.com/fod-dev/fod-uploader-java)
* [ScanCentral documentation¹](https://www.microfocus.com/documentation/fortify-software-security-center/2010/ScanCentral_Help_20.1.0/index.htm#CLI.htm%3FTocPath%3DFortify%2520ScanCentral%2520Command%2520Options%7C_____0)  
* [GitHub Action to set up ScanCentral Client](https://github.com/fortify/gha-setup-scancentral-client)
* [GitHub Workflow documentation](https://docs.github.com/en/actions/configuring-and-managing-workflows/configuring-a-workflow)
* Sample workflows:
    * [EightBall](https://github.com/fortify/gha-sample-workflows-eightball/tree/master/.github/workflows)
	* [SSC JS SandBox](https://github.com/fortify/gha-sample-workflows-ssc-js-sandbox/tree/master/.github/workflows)


¹ Note that in combination with FoD Uploader, *only* the ScanCentral `Package` command is relevant. Other ScanCentral commands are not used in combination with FoD Uploader, and none of the other ScanCentral components like ScanCentral Controller or ScanCentral Sensor are used when submitting scans to FoD.

### Considerations

* Be sure to consider the appropriate event triggers in your workflows, based on your project and branching strategy
* The environment variables and command line arguments in the example workflow can be modified to use API Key authentication instead of a Personal Access Tokens
* .NET payloads should not be packaged with ScanCentral Client, but rather must be packaged according to the Fortify on Demand guidelines. FoD support for .NET payloads generated by ScanCentral Client is currently in development.
* Windows-based runners use different syntax and different file locations. In particular:
    * Environment variables are referenced as `$Env:var` instead of `$var`, for example `"$Env:FOD_UPLOAD_JAR"` instead of `$FOD_UPLOAD_JAR`
    * ScanCentral logs are stored in a different location, so the upload-artifact step would need to be adjusted accordingly if you wish to archive ScanCentral logs
* If you are not already a Fortify customer, check out our [Free Trial](https://www.microfocus.com/en-us/products/application-security-testing/free-trial)


## Inputs

### `version`
**Required** The version of the Fortify Uploader to be set up. Default `latest`.

## Outputs

### `FOD_UPLOAD_JAR` environment variable
Specifies the location of the FoD Uploader JAR file
