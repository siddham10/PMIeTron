import groovy.json.JsonSlurper
import groovy.json.*
import java.io.OutputStreamWriter
import java.lang.String
import groovy.transform.Field
import jenkins.model.*
import java.io.FileReader;
import java.util.Iterator;
import java.util.Map
import groovy.io.FileType
import groovy.json.JsonSlurperClassic

def ZIP_NODE
def ZIP_WORKFLOW
def jsonResponse
String fileuploadUrl
String postuploadfileuploadUrl
def list = []
String taskID
String zipfilepath
String zip_workflowfilepath
String includefilenames
String filedata
def jsonIncludefilepath 
def postdeploymentfilepath
def statuscode
def objectstatus
def objectsecondstatus
String statusDetail1msg = ""
def status_final
def statusvalue
String includedfile = ""

pipeline {
    agent any

    stages {
        stage('Initialization...') {
            steps {
                script {
                    ZIP_NODE = "${env.BRANCH_NAME}.zip"
                    ZIP_WORKFLOW = "postdeployment.zip"
                    jsonIncludefilepath = "${env.WORKSPACE}/deployment-artifacts/includedfile.json"
                    zipfilepath = "${env.WORKSPACE}/${ZIP_NODE}"
                    zip_workflowfilepath = "${env.WORKSPACE}/${ZIP_WORKFLOW}"

                    println("ZIP_NODE: " + ZIP_NODE)
                    println("ZIP_WORKFLOW: " + ZIP_WORKFLOW)
                    println("jsonIncludefilepath: " + jsonIncludefilepath)
                    println("zipfilepath: " + zipfilepath)
                    println("zip_workflowfilepath: " + zip_workflowfilepath)
                }
            }
        }

        stage('prepare-package') {
            steps {
                script {
                    def includedFileObject = new File("${jsonIncludefilepath}")
                    def includedFileJSONObject = new JsonSlurperClassic().parse(includedFileObject)

                    def includedFilenamesString = includedFileJSONObject.filename

                    println("includedFilenamesString: " + includedFilenamesString)

                    if (includedFilenamesString.size() > 1) {
                        // The string contains a comma
                        for (int i = 0; i < includedFilenamesString.size(); i++) {
                            if (i >= 1) {
                                includedfile += ","
                            }
                            includedFilenamesString[i].trim().replaceAll("\\[|\\]", "")
                            includedfile += "'${env.WORKSPACE}" + "\\" + includedFilenamesString[i] + "'"
                        }
                        println("Final included files: " + includedfile)
                    } else {
                        // The string does not contain a comma
                        includedfile = includedFilenamesString
                        println("No comma present" + includedfile)
                    }

                    println("Final included files: " + includedfile)
                    println("Filenames processed successfully... Moving to zipping..")

                    bat """
                    powershell.exe -Command "if (Test-Path '${zipfilepath}') { Remove-Item '${zipfilepath}' }"
                    powershell.exe -Command "Compress-Archive -Path @(${includedfile}) -DestinationPath '${zipfilepath}'"
                    """

                    if (!fileExists(zipfilepath)) {
                        error("Failed to create the zip file.")
                    } else {
                        println("Files zipped successfully...")
                    }
                }
            }
        }
        
        stage('creating folder') {
            steps {
                script {
                    def jsonobject = "{\"binaryStreamObject\":{\"id\":\"guid\",\"type\":\"seedDataStream\",\"properties\":{\"objectKey\":\"${ZIP_NODE}\",\"originalFileName\":\"${ZIP_NODE}\"}}}"

                    def post = new URL("https://etronds.riversand.com/api/binarystreamobjectservice/prepareUpload").openConnection();
                    def message = '{"message":"this is a message"}'
                    post.setRequestMethod("POST")
                    post.setDoOutput(true)
                    post.setRequestProperty("Content-Type", "application/json")
                    post.setRequestProperty("x-rdp-version", "8.1")
                    post.setRequestProperty("x-rdp-tenantId", "etronds")
                    post.setRequestProperty("x-rdp-clientId", "rdpclient")
                    post.setRequestProperty("x-rdp-userId", "etronds.systemadmin@riversand.com")
                    post.setRequestProperty("x-rdp-userRoles", "systemadmin")
                    post.setRequestProperty("auth-client-id", "j29DTHa7m7VHucWbHg7VvYA75pUjBopS")
                    post.setRequestProperty("auth-client-secret", "J7UaRWQgxorI8mdfuu8y0mOLqzlIJo2hM3O4VfhX1PIeoa7CYVX_l0-BnHRtuSWB")
                    post.connect()

                    OutputStreamWriter out = new OutputStreamWriter(post.getOutputStream())
                    out.write(jsonobject)
                    out.close()
                    def statuscode1 = post.getResponseCode()
                    String outputObj = post.getInputStream().getText()
                    println("StatusCode=" + statuscode1)
                    println("outputObj=" + outputObj)
                    Map jsonContent = (Map) new JsonSlurper().parseText(outputObj)
                    String data1 = jsonContent.response.binaryStreamObjects.data
                    String[] arrOfStr = data1.split(",")
                    
                    for (int i = 0; i < arrOfStr.length; i++) {
                        String[] arrOfurl = arrOfStr[i].split("uploadURL=")
                        
                        for (int p = 1; p < arrOfurl.length; p++) {
                            fileuploadUrl = arrOfurl[p] - "}}]"
                            println("data[" + p + "]: " + arrOfurl[p])
                        }
                    }
                    
                    println(fileuploadUrl)
                }
            }
        }

        stage('Deploying folder') {
            steps {
                script {
                    echo "==== Deploying folder ===="

                    def encodedFileuploadUrl = fileuploadUrl.replaceAll('%', '%%')

                    bat """
                        curl -v -X PUT "${encodedFileuploadUrl}" ^
                        --header "x-ms-meta-x_rdp_userroles: systemadmin" ^
                        --header "x-ms-meta-x_rdp_tenantid: etronds" ^
                        --header "x-ms-meta-originalfilename: ${ZIP_NODE}" ^
                        --header "x-ms-blob-content-disposition: attachment; filename=${ZIP_NODE}" ^
                        --header "x-ms-meta-type: disposition" ^
                        --header "x-ms-meta-x_rdp_clientid: rdpclient" ^
                        --header "x-ms-meta-x_rdp_userid: etronds.systemadmin@riversand.com" ^
                        --header "x-ms-meta-binarystreamobjectid: guid" ^
                        --header "x-ms-blob-type: BlockBlob" ^
                        --header "Content-Type: application/zip" ^
                        --data-binary "@${zipfilepath}"
                    """
                }
            }
        }

        stage('Deployment into tenant'){
                steps{
                    script{
                        echo "====Deployment====="
                        def jsonobject = "{\"adminObject\":{\"id\":\"someguid\",\"type\":\"adminObject\",\"properties\":{\"flushConfig\":false,\"storageType\":\"stream\",\"objectKey\":\"${ZIP_NODE}\",\"tenantId\":\"etronds\",\"retryCount\":1,\"sleepTime\":1000}}}"
                        def post = new URL("https://etronds.riversand.com/api/adminservice/deploytenantseed").openConnection();
                        def message = '{"message":"this is a message"}'
                        post.setRequestMethod("POST")
                        post.setDoOutput(true)
                        post.setRequestProperty("Content-Type","application/zip")
                        post.setRequestProperty("x-rdp-version","8.1")
                        
                        post.setRequestProperty("x-rdp-clientId","rdpclient")
                        
                        post.setRequestProperty("x-rdp-userId","etronds.systemadmin@riversand.com")
                        
                        post.setRequestProperty("x-rdp-userRoles","systemadmin")
                        
                        post.setRequestProperty("auth-client-id","j29DTHa7m7VHucWbHg7VvYA75pUjBopS")
                        
                        post.setRequestProperty("auth-client-secret","J7UaRWQgxorI8mdfuu8y0mOLqzlIJo2hM3O4VfhX1PIeoa7CYVX_l0-BnHRtuSWB")
                        OutputStreamWriter out = new OutputStreamWriter(post.getOutputStream());
                        out.write(jsonobject);
                        out.close();
                        post.getOutputStream().write(message.getBytes("UTF-8"));
                        def statuscode2 = post.getResponseCode();
                        def outputObj = post.getInputStream().getText();
                        println("statusCode=" +statuscode2)
                        Map jsonContent = (Map) new JsonSlurper().parseText(outputObj)
                        println(jsonContent)
                        def status = jsonContent.response.status
                        def totalRecords= jsonContent.response.totalRecords
                        taskID = jsonContent.response.statusDetail.taskId
                        println("Status="+status);
                        println("Taskid="+taskID)
                        println("TotalRecod="+totalRecords);

                    }    
            }
        }

      

    }
}
