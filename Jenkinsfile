
def checkPublishStatus(String packageName, String packageVersion){
    def result = false
    
    npmViewScript = "npm view ${packageName}@${packageVersion} ${params.NPM_REGISTRY}"
    echo "** Ayn? versiyon kullan?lm?? m? kontrolü için script > npmViewScript: ${npmViewScript}"
    /* E?er npm view aranan paketi bulamazsa sh komutu 1 (Error 404) ile hata f?rlatarak ç?kacak!
     * Bu yüzden kod k?r?lmas?n diye "returnStatus: true" ile sh execute edilecek ve exit code okunacak.
     * Exit code 0'dan farkl?ysa ise yani "npm ERR! code E404" ile npm view hata f?rlatarak ç?k?? yapm??sa bilece?iz ki; paket yok!
     * E?er normal ç?k?? yapm??sa bu kez ç?kt?y? almak için "returnStdout: true" anahtar?yla tekrar paket sorgulanacak
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
                label: "Paket yüklü de?il ve yay?nlans?n istendi?i için Publishing",
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
        // zorla yay?nla denmi?se ve paket yay?ndaysa kald?rmal?y?z
        if(isPublished)
        {
            unpublish packageName, packageVersion
        }
    }
}

def checkPublishable(Boolean isPublished){
    if( params.FORCE_TO_PUBLISH == false){
        
        if(params.PUBLISH_IF_NOT && isPublished){
            // zorla publish etme seçene?i i?aretli de?ilse ve yay?nlanmas? isteniyorsa, yay?nlanm?? olmas? durumunda hata f?rlataca??z. 
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

def checkout_cem(String url, String branch="master", String credId){
    echo "url:${url}, branch:${branch}, credId:${credId}"
    git branch: branch, credentialsId: credId, url: url
}

def installPackages(){
    kapsam = ["@kapsam1","@kapsam2"]
    nodemodules_folder_path = "${WORKSPACE}/node_modules"
    echo "nodemodules_folder_path: ${nodemodules_folder_path}"
    is_nodemodules_exits = fileExists(nodemodules_folder_path)
    echo "is_nodemodules_exits: ${is_nodemodules_exits}"
    if( is_nodemodules_exits == false){
        echo "*** NODE_MODULES Yok! NPM paketlerini yükleyece?iz"
        for(i=0; i<kapsam.size(); i++) {
            scope = kapsam[i]
            sh "npm config set ${scope}:registry ${params.NPM_REGISTRY.replace('--registry=','')} "
        }
        //sh "npm --cache-min Infinity install"
        sh "pwd && npm install ${params.NPM_REGISTRY}"
    }else{
        echo "*** NODE_MODULES var ve tekrar NPM paketlerini yüklemeyelim"
    }
}
                    
pipeline {
	agent { label params.AGENT_NAME }
	
    parameters {
        string(trim: true, name: 'AGENT_NAME', defaultValue: 'docker_slave', description: 'Hangi slave üstünde çal??aca?? bilgisi')
        string(trim: true, name: 'GIT_HTTPS_CRED_ID', defaultValue: 'f483b6a5-1204-41d9-a82e-000d495fe34b', description: 'HTTPs ile ba?lanaca?? user id')
        string(trim: true, name: 'GIT_CRED_ID', defaultValue: 'github-user-pass-cemtopkaya', description: 'GIT Repo ba?lant?s? olacaksa CRED_ID kullan?lacak')
        string(trim: true, name: 'SOURCE_BRANCH_NAME', defaultValue: 'developer', description: 'Kodlar? hangi BRANCH üstünden çekece?ini belirtiyoruz')
        string(trim: true, name: 'TARGET_BRANCH_NAME', defaultValue: 'master', description: 'Push ile kodun gönderilece?i branch')

        text(name: 'REPOS', defaultValue: 'https://github.com/cemtopkaya/jenkins-shared-lib-project-multi-repo-angular-lib-1.git\nhttps://github.com/cemtopkaya/jenkins-shared-lib-project-multi-repo-angular-lib-2.git', description: 'Kütüphanelerin reposu')
        
        booleanParam(name: 'FORCE_TO_PUBLISH', defaultValue: false, description: 'E?er versiyon daha önce kullan?lm??sa zorla ayn? versiyon numaras?yla VERDACCIO ya yay?nlar ')
        booleanParam(name: 'PUBLISH_IF_NOT', defaultValue: false, description: 'Daha önce yay?nlanmam??sa yay?nla, aksi halde hata f?rlat ')
        booleanParam(name: 'CLEAN_WORKSPACE', defaultValue: true, description: 'WorkSpace i temizle')
        booleanParam(name: 'RUN_PARALLEL', defaultValue: false, description: 'Paralel çal??t?r')

        choice(
            name: 'NPM_REGISTRY', 
            choices: [
                ' --registry=http://192.168.56.1:4873 ',
                ' --registry=http://localhost:4873 ',
                ' --registry=http://192.168.13.183:4873 '
            ], 
            description: '')
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

        stage("checkout repos"){
            steps{

                echo "====++++executing checkout repos++++===="
                echo "params.REPOS: ${REPOS}"
                
				script {
                    repos = params.REPOS.split("\n")
                    for(i=0;i<repos.size();i++){
                        repo = repos[i]
                        echo "repo adresi: ${repo}"
                        checkout_cem (repo, params.SOURCE_BRANCH_NAME, params.GIT_CRED_ID)
                        installPackages()
                    }
                }
            }
        }

		// stage("Checkout"){
		    
		//     environment{
        //         PACKAGE_FILE_PATH = "${WORKSPACE}/package.json"
        //         IS_PACKAGE_FILE_EXISTS = fileExists PACKAGE_FILE_PATH
		//     }
		    
		// 	steps {
		// 		script {
        //             def cmd = "cat angular.json | grep '\"projectType\": \"library\"' -C1 | grep -oE '\"(@.*|projects.*)>?\"'"
        //             def errors = sh( returnStdout: true, script: cmd )
        //             def list = errors.split( '\n' )
                    
        //             println "eLise hali: ${list}"
                
    	// 	        if(env.IS_PACKAGE_FILE_EXISTS.toBoolean() == false) {
        // 				echo "** Dosyalar olmad??? için SCM'den checkout yapal?m"
        // 			   //git branch: params.SOURCE_BRANCH_NAME, credentialsId: params.GIT_HTTPS_CRED_ID, url: params.GIT_REPO_ADDR_HTTPS
        // 			    git branch: params.SOURCE_BRANCH_NAME, credentialsId: params.GIT_SSH_CRED_ID, url: params.GIT_REPO_ADDR_SSH
    	// 	        }
		// 		}
		// 	}
		// }
		
		// stage("Install NPM Packages"){
        //     environment {
        //         NODEMODULES_FOLDER_PATH = "${WORKSPACE}/node_modules"
        //         IS_NODEMODULES_EXISTS = fileExists(NODEMODULES_FOLDER_PATH)
        //     }
		// 	steps {
		// 	    script{
		// 	        if( env.IS_NODEMODULES_EXISTS == "false"){
        // 			    echo "*** NODE_MODULES Yok! NPM paketlerini yükleyece?iz"
        // 			    sh "npm config set @cinar:registry ${params.NPM_REGISTRY.replace('--registry=','')} "
        // 			    sh "npm --cache-min Infinity install"
		// 	        }else{
        // 			    echo "*** NODE_MODULES var ve tekrar NPM paketlerini yüklemeyelim"
		// 	        }
		// 	    }
		// 	}
		// }

		// stage("Find Package Version"){
        //     environment { IS_PUBLISHED=false }
            
		// 	steps {
        //         script {
    	// 	        names = params.SCOPED_PACKAGE_NAMES.split("\n")
    	// 	        paths = params.PACKAGE_SOURCE_PATHS.split("\n")
    		        
    	// 	        if(names.size() != names.size()) {
    	// 	            error('Parametrelerde verilen paket isimleriyle paket dosya yollar? ayn? say?da de?il! Yap?land?rma iptal edildi!')
    	// 	        }
                        		        
    	// 	       def publish = [:]
    	// 	       for(i=0;i<names.size();i++){
    	// 	           echo "i: ${i instanceof Integer}"
    	// 	            def name = names[i]
        // 		        def path = paths[i]
        		       
        // 		        if(params.RUN_PARALLEL == false) {
        // 		          oneNode(name, path)
        // 		        }else{
        // 		            publish["Publish ${name}"] = { oneNode(name, path) }
        // 		        }
    	// 	       }
    		       
    	// 	       if(params.RUN_PARALLEL) {
    	// 	           parallel publish
    	// 	       }
    	// 	    }
		// 	}
		// }
		
		stage('Git Push To Origin') {
            steps {
                script {
                    return null
                    tagName = "${env.JOB_NAME}-v${MAJOR}.${PHASE_NUMBER}.${SPRINT_NUMBER}-b${env.BUILD_NUMBER}"
                    eposta = "jenkins.service@ulakhaberlesme.com.tr"
                    name = "Jenkins Servis"
                    
                    echo "*** Etiketlenecek ve Push edilecek. Kullan?lacak etikat ad?: ${tagName}"
                    
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
            echo "** Süreç ba?ar?yla tamamland?"
        }
        failure {
            echo "** Süreç hatal? tamamland?"
        }
        cleanup{
            echo "** Süreç tamamland? cleanup zaman?"
        }
        always { 
            echo "** Süreç günah?yla sevab?yla tamamland?"
        }
    }

}