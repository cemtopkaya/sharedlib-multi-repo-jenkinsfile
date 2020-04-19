def checkPublishStatus(String packageName, String packageVersion){
    def result = false
    
    npmViewScript = "npm view ${packageName}@${packageVersion} ${params.NPM_REGISTRY}"
    echo "** Aynı versiyon kullanılmış mı kontrolü için script > npmViewScript: ${npmViewScript}"
    /* Eğer npm view aranan paketi bulamazsa sh komutu 1 (Error 404) ile hata fırlatarak çıkacak!
     * Bu yüzden kod kırılmasın diye "returnStatus: true" ile sh execute edilecek ve exit code okunacak.
     * Exit code 0'dan farklıysa ise yani "npm ERR! code E404" ile npm view hata fırlatarak çıkış yapmışsa bileceğiz ki; paket yok!
     * Eğer normal çıkış yapmışsa bu kez çıktıyı almak için "returnStdout: true" anahtarıyla tekrar paket sorgulanacak
     **/
    
    npmViewStatusCode = sh(returnStatus: true, script: "$npmViewScript")
    if(npmViewStatusCode == 1){
        result = false
    }else {
        checkVersionPublished = sh(
            label: "$npmViewScript",
            returnStdout: true, 
            script: "${npmViewScript} | wc -m"
        ).trim() as Integer
        
        result = checkVersionPublished > 0
    }

    echo "is published: $result"
    return result
}

def buildPackage(String packageName){
    if( params.PUBLISH_IF_NOT == true || params.FORCE_TO_PUBLISH == true){
        sh (
            label:"NPM Package Building ($packageName)",
            script: "ng build $packageName"
        )
    }
}

def publishIfNeeded(packageName, packageSrcPath, packageVersion, Boolean isPublished){
        		        
    echo "NPM Package Publishing ($packageName)"
    
    dir("${WORKSPACE}${packageSrcPath.replace('projects', 'dist')}"){
        unpublish(packageName, packageVersion)
        
        def shStatusCode = 0
        if ( params.FORCE_TO_PUBLISH == true) {
            shStatusCode = sh (
                label: "Zorla Publishing",
                returnStatus: true,
                script: "npm publish ${params.NPM_REGISTRY} --force"
            )
        } else if (params.PUBLISH_IF_NOT == true && isPublished == false) {
            shStatusCode = sh (
                label: "Paket yüklü değil ve yayınlansın istendiği için Publishing",
                returnStatus: true,
                script: "npm publish ${params.NPM_REGISTRY}"
            )
        }
        
        checkPublishStatus(packageName, packageVersion)
    }
}

def unpublish(packageName, packageVersion){
    sh (
        label:"Unpublish Package: ${packageName}@${packageVersion}",
        script: "npm unpublish ${packageName}@${packageVersion}  ${params.NPM_REGISTRY}"
    )
}

def unpublishIfNeeded(String packageName, String packageVersion, Boolean isPublished){
    if( params.FORCE_TO_PUBLISH || params.PUBLISH_IF_NOT ){
        // zorla yayınla denmişse ve paket yayındaysa kaldırmalıyız
        if(isPublished)
        {
            unpublish packageName, packageVersion
        }
    }
}

def checkPublishable(Boolean isPublished){
    if( params.FORCE_TO_PUBLISH == false){
        
        if(params.PUBLISH_IF_NOT && isPublished){
            // zorla publish etme seçeneği işaretli değilse ve yayınlanması isteniyorsa, yayınlanmış olması durumunda hata fırlatacağız. 
            error('Package has been published before! Aborting the build.')
        }
    }
    
    return isPublished == false
}

def getPackageVersion(packageSrcPath){
    String version = sh(returnStdout: true, script: "node -p -e \"require(\'${WORKSPACE}${packageSrcPath}/package.json\').version\"").trim()
    echo "** Paketin versiyonu > $version"
    version
}


def oneNode = { name, path ->
    echo ">>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>"
    echo "---->$i - name: ${name} - path: ${path}"
    
    packageVersion = getPackageVersion path
    
    Boolean isPublished = checkPublishStatus(name, packageVersion)
    
    Boolean isPublishable = checkPublishable(isPublished)
    
    unpublishIfNeeded(name, packageVersion, isPublished)
    
    buildPackage name
    
    publishIfNeeded name, path, packageVersion, isPublished
    echo "<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<"
}
                    
pipeline {
	agent { label params.AGENT_NAME }
	
    parameters {
        string(name: 'AGENT_NAME', defaultValue: 'docker_slave', description: 'Hangi slave üstünde çalışacağı bilgisi')
        string(name: 'GIT_HTTPS_CRED_ID', defaultValue: 'f483b6a5-1204-41d9-a82e-000d495fe34b', description: 'HTTPs ile bağlanacağı user id')
        string(name: 'GIT_REPO_ADDR_SSH', defaultValue: 'ssh://git@bitbucket.ulakhaberlesme.com.tr:7999/cin/gui_lib_test.git', description: 'REPO ya SSH protokolü üstünden bağlanacaksak bu alan boş bırakılmamalı')
        string(name: 'GIT_SSH_CRED_ID', defaultValue: 'a64a70a5-6e93-4afe-9bab-aff1ddc1b9d3', description: 'GIT Repo bağlantısı SSH protokolüyle olacaksa, SSH Key bilgisi içeren CRED_ID kullanılacak')
        string(name: 'SOURCE_BRANCH_NAME', defaultValue: 'developer', description: 'Kodları hangi BRANCH üstünden çekeceğini belirtiyoruz')
        string(name: 'TARGET_BRANCH_NAME', defaultValue: 'master', description: 'Push ile kodun gönderileceği branch')

        text(name: 'SCOPED_PACKAGE_NAMES', defaultValue: '', description: 'Enter some information about the person')

        booleanParam(name: 'FORCE_TO_PUBLISH', defaultValue: true, description: 'Eğer versiyon daha önce kullanılmışsa zorla aynı versiyon numarasıyla VERDACCIO ya yayınlar ')
        booleanParam(name: 'PUBLISH_IF_NOT', defaultValue: true, description: 'Daha önce yayınlanmamışsa yayınla, aksi halde hata fırlat ')
        booleanParam(name: 'CLEAN_WORKSPACE', defaultValue: false, description: 'WorkSpace i temizle')
        booleanParam(name: 'RUN_PARALLEL', defaultValue: false, description: 'Paralel çalıştır')

        choice(name: 'NPM_REGISTRY', choices: [' --registry=http://192.168.13.183:4873 ', ' --registry=http://localhost:4873 '], description: '')
        choice(name: 'NPM_REGISTRY1', choices: [' --registry=http://192.168.13.183:4873 ', ' --registry=http://localhost:4873 '], description: '')
    }
	
	stages {
	    
		stage("Clean Workspace"){
            when {
                expression { params.CLEAN_WORKSPACE as Boolean == true }
            }
			steps {
				echo "*** Klasörü temizleyelim"
			    cleanWs()
			}
		}

		stage("Checkout"){
		    
		    environment{
                PACKAGE_FILE_PATH = "${WORKSPACE}/package.json"
                IS_PACKAGE_FILE_EXISTS = fileExists PACKAGE_FILE_PATH
		    }
		    
			steps {
				script {
                    def cmd = "cat angular.json | grep '\"projectType\": \"library\"' -C1 | grep -oE '\"(@.*|projects.*)>?\"'"
                    def errors = sh( returnStdout: true, script: cmd )
                    def list = errors.split( '\n' )
                    
                    println "eLise hali: ${list}"
                
    		        if(env.IS_PACKAGE_FILE_EXISTS.toBoolean() == false) {
        				echo "** Dosyalar olmadığı için SCM'den checkout yapalım"
        			   //git branch: params.SOURCE_BRANCH_NAME, credentialsId: params.GIT_HTTPS_CRED_ID, url: params.GIT_REPO_ADDR_HTTPS
        			    git branch: params.SOURCE_BRANCH_NAME, credentialsId: params.GIT_SSH_CRED_ID, url: params.GIT_REPO_ADDR_SSH
    		        }
				}
			}
		}
		
		stage("Install NPM Packages"){
            environment {
                NODEMODULES_FOLDER_PATH = "${WORKSPACE}/node_modules"
                IS_NODEMODULES_EXISTS = fileExists(NODEMODULES_FOLDER_PATH)
            }
			steps {
			    script{
			        if( env.IS_NODEMODULES_EXISTS == "false"){
        			    echo "*** NODE_MODULES Yok! NPM paketlerini yükleyeceğiz"
        			    sh "npm config set @cinar:registry ${params.NPM_REGISTRY.replace('--registry=','')} "
        			    sh "npm --cache-min Infinity install"
			        }else{
        			    echo "*** NODE_MODULES var ve tekrar NPM paketlerini yüklemeyelim"
			        }
			    }
			}
		}

		stage("Find Package Version"){
            environment { IS_PUBLISHED=false }
            
			steps {
                script {
    		        names = params.SCOPED_PACKAGE_NAMES.split("\n")
    		        paths = params.PACKAGE_SOURCE_PATHS.split("\n")
    		        
    		        if(names.size() != names.size()) {
    		            error('Parametrelerde verilen paket isimleriyle paket dosya yolları aynı sayıda değil! Yapılandırma iptal edildi!')
    		        }
                        		        
    		       def publish = [:]
    		       for(i=0;i<names.size();i++){
    		           echo "i: ${i instanceof Integer}"
    		            def name = names[i]
        		        def path = paths[i]
        		       
        		        if(params.RUN_PARALLEL == false) {
        		          oneNode(name, path)
        		        }else{
        		            publish["Publish ${name}"] = { oneNode(name, path) }
        		        }
    		       }
    		       
    		       if(params.RUN_PARALLEL) {
    		           parallel publish
    		       }
    		    }
			}
		}
		
		stage('Git Push To Origin') {
            steps {
                script {
                    return null
                    tagName = "${env.JOB_NAME}-v${MAJOR}.${PHASE_NUMBER}.${SPRINT_NUMBER}-b${env.BUILD_NUMBER}"
                    eposta = "jenkins.service@ulakhaberlesme.com.tr"
                    name = "Jenkins Servis"
                    
                    echo "*** Etiketlenecek ve Push edilecek. Kullanılacak etikat adı: ${tagName}"
                    
                    sshagent([params.GIT_SSH_CRED_ID]) {
                        sh "git config --local user.email '${eposta}'"
                        sh "git config --local user.name '$name'"
                        sh "git tag -fa ${tagName} -m 'git tag oldu'"
                        sh "git push origin HEAD:$TARGET_BRANCH_NAME --tags"
                    }
                }
            }
        }
	}
	
	
    post { 
        success{
            echo "** Süreç başarıyla tamamlandı"
        }
        failure {
            echo "** Süreç hatalı tamamlandı"
        }
        cleanup{
            echo "** Süreç tamamlandı cleanup zamanı"
        }
        always { 
            echo "** Süreç günahıyla sevabıyla tamamlandı"
        }
    }

}