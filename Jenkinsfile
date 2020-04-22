
@Library('sl-multi-repo')

import Parser.AngularParser;
import Sorter.BuildSorter; 
import hudson.FilePath;

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
        try {
            checkVersionPublished = sh(
                label: "$npmViewScript",
                returnStdout: true, 
                script: "${npmViewScript} | wc -m"
            ).trim() as Integer
            
            result = checkVersionPublished > 0
        }
        catch (err) {
            println "npm view hata fırlattı: $err"
        }
    }

    echo "is published: $result"
    return result
}

def buildPackage(String packageName){
    echo "-> ------------- buildPackage ------------------"
    if( params.PUBLISH_IF_NOT == true || params.FORCE_TO_PUBLISH == true){
        try {
            sh (
                label:"NPM Package Building ($packageName)",
                returnStatus: false,
                script: "ng build $packageName"
            )
        }
        catch (err) {
            println "npm view hata fırlattı: $err"
        }
    }
}

def publishIfNeeded(packageName, packageSrcPath, packageVersion, Boolean isPublished){
        		        
    echo "NPM Package Publishing ($packageName)"
    def libDistFolderPath = "${packageSrcPath.replace('projects', 'dist')}"
    dir(libDistFolderPath){
        unpublish(packageName, packageVersion)
        
        def shStatusCode = 0
        if ( params.FORCE_TO_PUBLISH == true) {
            try {
                shStatusCode = sh (
                    label: "Zorla Publishing: $libDistFolderPath",
                    returnStatus: true,
                    script: "npm publish ${params.NPM_REGISTRY} --force"
                )
            }
            catch (err) {
                println "npm publish hata fırlattı: $err"
            }
        } else if (params.PUBLISH_IF_NOT == true && isPublished == false) {
            try {                
                shStatusCode = sh (
                    label: "Paket yüklü değil ve yayınlansın istendiği için Publishing: $libDistFolderPath",
                    returnStatus: true,
                    script: "npm publish ${params.NPM_REGISTRY}"
                )
            }
            catch (err) {
                println "npm publish else içinde hata fırlattı: $err"
            }
        }
        
        checkPublishStatus(packageName, packageVersion)
    }
}

def unpublish(packageName, packageVersion){
    try {
        sh (
            label:"Unpublish Package: ${packageName}@${packageVersion}",
            returnStatus: false,
            script: "npm unpublish ${packageName}@${packageVersion}  ${params.NPM_REGISTRY}"
        )
    }
    catch (err) {
        println "npm unpublish içinde hata fırlattı: $err"
    }
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
    def json = readJSON(file: "$packageSrcPath/package.json")
    
    String version = json.version
    echo "** Paketin versiyonu > $version"
    version
}


def oneNode = { name, path ->
    echo ">>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>"
    echo "---->$i - name: $name - path: $path"
    
    packageVersion = getPackageVersion path
    
    Boolean isPublished = checkPublishStatus(name, packageVersion)
    
    Boolean isPublishable = checkPublishable(isPublished)
    
    unpublishIfNeeded(name, packageVersion, isPublished)
    
    buildPackage name
    
    publishIfNeeded name, path, packageVersion, isPublished
    echo "<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<"
}

def installPackages(String sourceFolder){
    is_nodemodules_exits = fileExists("node_modules")
    echo "-> is_nodemodules_exits: $is_nodemodules_exits"

    if( is_nodemodules_exits == false){
        echo "*** NODE_MODULES Yok! NPM paketlerini yükleyeceğiz"
        sh "npm config set registry ${params.NPM_REGISTRY.replace('--registry=','')} "
        //sh "npm --cache-min Infinity install"
        sh "pwd && npm install --no-bin-links"
    }else{
        echo "*** NODE_MODULES var ve tekrar NPM paketlerini yüklemeyelim"
    }
}

def npmLogin(_userName, _pass, _email, _registry){
    userName = _userName || "jenkins.service"
    pass = _pass || "cicd123"
    email = _email || "test@example.com"
    registry = _registry || "http://192.168.56.1:4873"
    cikti = sh (
        label: "npm-cli-login ile Login oluyoruz",
        script: " npm-cli-login -u $userName -p $pass -e $email -r $registry",
        returnStdout: true
    )

}

def installNpmCliLogin(){
    sh "npm install -g npm-cli-login"
}

pipeline {
	agent { label params.AGENT_NAME }
	
    parameters {
        string(trim: true, name: 'AGENT_NAME', defaultValue: 'docker_slave', description: 'Hangi slave Üstünde çalışacağı bilgisi')
        string(trim: true, name: 'GIT_HTTPS_CRED_ID', defaultValue: 'f483b6a5-1204-41d9-a82e-000d495fe34b', description: 'HTTPs ile bağlanacağı user id')
        string(trim: true, name: 'GIT_CRED_ID', defaultValue: 'github-user-pass-cemtopkaya', description: 'GIT Repo bağlantısı olacaksa CRED_ID kullanılacak')
        string(trim: true, name: 'SOURCE_BRANCH_NAME', defaultValue: 'developer', description: 'Kodları hangi BRANCH üstünden çekeceğini belirtiyoruz')
        string(trim: true, name: 'TARGET_BRANCH_NAME', defaultValue: 'master', description: 'Push ile kodun gönderileceği branch')


        string(trim: true, name: 'NPM_USERNAME', defaultValue: 'jenkins.service', description: 'NPM Kullanıcı Bilgileri')
        string(trim: true, name: 'NPM_PASS', defaultValue: 'q1w2e3r4', description: 'NPM Kullanıcı Bilgileri')

        text(name: 'REPOS', defaultValue: 'https://github.com/cemtopkaya/jenkins-shared-lib-project-multi-repo-angular-lib-2.git', description: 'Kütüphanelerin reposu')
        // text(name: 'REPOS', defaultValue: 'https://github.com/cemtopkaya/jenkins-shared-lib-project-multi-repo-angular-lib-1.git\nhttps://github.com/cemtopkaya/jenkins-shared-lib-project-multi-repo-angular-lib-2.git', description: 'Kütüphanelerin reposu')
        
        booleanParam(name: 'FORCE_TO_PUBLISH', defaultValue: true, description: 'Eğer versiyon daha önce kullanılmışsa zorla aynı versiyon numarasıyla VERDACCIO ya yayınlar ')
        booleanParam(name: 'PUBLISH_IF_NOT', defaultValue: false, description: 'Daha önce yayınlanmamışsa yayınla, aksi halde hata fırlat ')
        booleanParam(name: 'CLEAN_WORKSPACE', defaultValue: false, description: 'WorkSpace i temizle')
        booleanParam(name: 'RUN_PARALLEL', defaultValue: false, description: 'Paralel çalıştır')

        choice(
            name: 'NPM_REGISTRY', 
            choices: [
                ' --registry=http://192.168.56.1:4873 ',
                ' --registry=http://192.168.13.183:4873 ',
                ' --registry=http://localhost:4873 '
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

        stage('PreRequisites'){
            steps{
                script{

                    try {
                        is_node_installed = sh(
                            label: "NODE Yüklü mü?",
                            returnStdout: true, 
                            script: "whereis node | grep ' ' -ic"
                        ).trim() as Integer
                    }
                    catch (exception) {
                        sh(
                            label: "NodeJs Yükleniyor",
                            returnStdout: false, 
                            script: "sudo apt-get update                                           \
                                        && apt-get upgrade -y                                      \
                                        && curl -sL https://deb.nodesource.com/setup_12.x | bash - \
                                        && apt-get install -y nodejs                               \
                                        && node --version                                          \
                                        && npm --version"
                        )
                    }

                    try {
                        is_angular_cli_installed =  sh(
                            label: "Angular CLI Yüklü mü?",
                            returnStdout: true, 
                            script: "ng --version"
                        )
                    }
                    catch (err) {
                            //script: "ng --version | grep '8.3.23' -i -c"
                        sh(
                            label: "Angular CLI Yükleniyor",
                            returnStdout: false, 
                            script: "npm install -g @angular/cli@8.3.23"
                        )
                    }
                }
            }
        }

        stage('Npm Login'){
            steps{
                script{
                    try {
                        npmLogin(
                            _userName:params.NPM_USERNAME, 
                            _pass:params.NPM_PASS, 
                            _email:"test@example.com", 
                            _registry:params.NPM_REGISTRY.replace('--registry=','').trim()
                        )
                    }
                    catch (err) {
                        echo "-> Hata:   $err"
                        installNpmCliLogin()
                        npmLogin(
                            _userName:params.NPM_USERNAME, 
                            _pass:params.NPM_PASS, 
                            _email:"test@example.com", 
                            _registry:params.NPM_REGISTRY.replace('--registry=','').trim()
                        )
                    }
                }
            }
        }

        stage("checkout repos"){
            steps{

                echo "====++++executing checkout repos++++===="
                echo "params.REPOS: ${REPOS}"
                
				script {
                    dirSourceCode = "./source_codes"
                    repos = params.REPOS.split("\n")
                    for(i=0;i<repos.size();i++){
                       
                            try {

                                repo = repos[i]
                                echo "-> repo adresi:  ${repo}"
                                def projectPath = "$dirSourceCode"

                                dir(projectPath){
                                    checkoutSCM(repo, params.SOURCE_BRANCH_NAME, params.GIT_CRED_ID)
                                    installPackages(".")
                                    def projectLibs = getLibs(".")
                                    echo "-> projectLibs: $projectLibs"
                                    println "-> ------------ getLibDependencies ---------"
                                    projectLibs.each{
                                        println it.value.path
                                        /**
                                        * ./projects içindeki kütüphanelerin bağımlılıklarını bulalım 
                                        */
                                        
                                        // ./projects/@kapsam/kütüp_adı yolunu olusturalım
                                        def libDirPath = "./$it.value.path"

                                        // paketin bağımlılıklarını bulalım
                                        it.value.dependencies  = getLibDependencies(libDirPath)
                                    }

                            
                                    println "-> ------------ getSortedLibraries ---------"
                                    // Tüm bağımlılıkları en az bağımlıdan, en çoka doğru sıralayalım
                                    def sortedLibs = getSortedLibraries(projectLibs)

                                    sortedLibs.each{ libName ->
                                        println "Kütüp adı: $libName"
                                        def paket = projectLibs."$libName"
                                        println "Paketttttttttt: $paket"
                                        def libPath = "./$paket.path"
                                        println "LibPathhhhh: $libPath"
                                        oneNode(libName, libPath)
                                    }
                                    // println ">>>>>>> Sorted Libs: $map"
                                    // println ">>>>>>> Sorted Deps: $res"
                                
                                }
                            }
                            catch (err) {
                                println "!!!!!!!!!!! istisna !!!!!!!!!!!!!!"
                                echo "Caught: $err"
                            } 

                        // res.each { entry ->
                        //     println entry.value.dependencies
                        // } 
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
        // 				echo "** Dosyalar olmadığı için SCM'den checkout yapalım"
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
        // 			    echo "*** NODE_MODULES Yok! NPM paketlerini yükleyeceğiz"
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
    	// 	            error('Parametrelerde verilen paket isimleriyle paket dosya yolları aynı sayıda değil! Yapılandırma iptal edildi!')
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
		
		// stage('Git Push To Origin') {
        //     steps {
        //         script {
        //             // return null
        //             // tagName = "${env.JOB_NAME}-v${MAJOR}.${PHASE_NUMBER}.${SPRINT_NUMBER}-b${env.BUILD_NUMBER}"
        //             // eposta = "jenkins.service@ulakhaberlesme.com.tr"
        //             // name = "Jenkins Servis"
                    
        //             // echo "*** Etiketlenecek ve Push edilecek. Kullanılacak etikat adı: ${tagName}"
                    
        //             // sshagent([params.GIT_SSH_CRED_ID]) {
        //             //     sh "git config --local user.email '${eposta}'"
        //             //     sh "git config --local user.name '$name'"
        //             //     sh "git tag -fa ${tagName} -m 'git tag oldu'"
        //             //     sh "git push origin HEAD:$TARGET_BRANCH_NAME --tags"
        //             // }
        //         }
        //     }
        // }
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