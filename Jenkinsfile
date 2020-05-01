@Library('gui_multi_repo@master')

import Parser.AngularParser;
import Sorter.BuildSorter; 
import hudson.FilePath;
import com.cloudbees.groovy.cps.NonCPS
import java.util.LinkedHashMap

def checkPublishStatus(String packageName, String packageVersion, String registry){
    def result = false
    result = isPackagePublished(scheme_host_port, packageName, packageVersion)
    return result
}

def buildPackage(String packageName){
    println "----------------- buildPackage -----------------"
    sh "pwd"
    if( params.PUBLISH_IF_NOT == true || params.FORCE_TO_PUBLISH == true){
        catchError (buildResult: "SUCCESS", stageResult:"UNSTABLE"){
            sh (
                label:"NPM Package Building ($packageName)",
                returnStatus: false,
                script: "ng build $packageName"
            )
        }
    }
}

//@NonCPS
def publishIfNeeded(packageName, packageSrcPath, packageVersion, Boolean isPublished){
    println "----------------- publishIfNeeded -----------------"
		        
    echo "NPM Package Publishing ($packageName)"
    def libDistFolderPath = packageSrcPath.replace('projects', 'dist')
    dir(libDistFolderPath){
        unpublish(packageName, packageVersion)
        
        def shStatusCode = 0
        try {
            def label = ""
            def script = ""

            if (params.FORCE_TO_PUBLISH == true) {
                label = "Zorla Publishing: $libDistFolderPath"
                script = "npm publish ${params.NPM_REGISTRY} --force"
            } else if (params.PUBLISH_IF_NOT == true && isPublished == false) {
                label = "Paket yüklü değil ve yayınlansın istendiği için Publishing: $libDistFolderPath"
                script = "npm publish ${params.NPM_REGISTRY}"
            }
            
            shStatusCode = sh (
                label: label,
                script: script,
                returnStatus: true
            )
        }
        catch (err) {
            println "---*** Hata (publishIfNeeded): $script çalıştırılırken istisna oldu (Exception: $err)"   
        }
        
        
        def registry = params.NPM_REGISTRY.replace("--registry=","").trim()
        checkPublishStatus(packageName, packageVersion, registry)
    }
}

def unpublish(packageName, packageVersion){
    println "----------------- getPackageVersion -----------------"

    def label = "Unpublish Package: ${packageName}@${packageVersion}"
    def script = "npm unpublish ${packageName}@${packageVersion}  ${params.NPM_REGISTRY}"
    try {
        sh (
            label: label,
            returnStatus: false,
            script: script
        )
    }
    catch (err) {
        println "---*** Hata (unpublish): $script çalıştırılırken istisna oldu (Exception: $err)"   
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
    println "----------------- checkPublishable -----------------"

    if( params.FORCE_TO_PUBLISH == false){
        
        if(params.PUBLISH_IF_NOT && isPublished){
            // zorla publish etme seçeneği işaretli değilse ve yayınlanması isteniyorsa,
            // yayınlanmış olması durumunda hata fırlatacağız. 
            //error('Package has been published before! Aborting the build.')
        }
    }
    
    return isPublished == false
}

def getPackageVersion(packageSrcPath){
    println "----------------- getPackageVersion -----------------"

    def packageJsonPath = "$packageSrcPath/package.json"
    try {
        def json = readJSON(file: packageJsonPath)
        
        String version = json.version
        echo "---*** Paketin versiyonu > $version"
        return version
    }
    catch (err) {
        println "---*** Hata (getPackageVersion): $packageJsonPath yolu readJSON ile okurken istisna oldu (Exception: $err)"  
    }
}

def oneNode(name, path){
    println "----------------- oneNode -----------------"
    echo "---->name: $name - path: $path"
    dir(path){
        packageVersion = getPackageVersion "$path"
        
        def registry = params.NPM_REGISTRY.replace("--registry=","").trim()
        Boolean isPublished = checkPublishStatus(name, packageVersion, registry)
        
        if(isPublished){ currentBuild.result = "UNSTABLE" }
        
        Boolean isPublishable = checkPublishable(isPublished)
        
        unpublishIfNeeded(name, packageVersion, isPublished)
        
        buildPackage name
        
        publishIfNeeded name, path, packageVersion, isPublished
    }
}

def installPackages(String sourceFolder){
    println ">>> sourceFolder: "+sourceFolder
    dir(sourceFolder){
        is_nodemodules_exits = fileExists("node_modules")
        echo "---*** is_nodemodules_exits: $is_nodemodules_exits"

        if( is_nodemodules_exits == false){
            echo "*** NODE_MODULES Yok! NPM paketlerini yükleyeceğiz"
            sh "npm config set registry ${params.NPM_REGISTRY.replace('--registry=','')} "
            //sh "npm --cache-min Infinity install"
            sh "pwd && npm install --no-bin-links"
        }else{
            echo "*** NODE_MODULES var ve tekrar NPM paketlerini yüklemeyelim"
        }
    }
}

def npmLogin(_userName, _pass, _email="jenkins@service.com", _registry){
    echo "_userName: $_userName, _pass: $_pass, _email: $_email, _registry: $_registry"
    userName = _userName ?: "jenkins"
    pass = _pass ?: "service"
    email = _email ?: "jenkins@service.com"
    registry = _registry ?: params.NPM_REGISTRY.replace('--registry=','')
    echo "userName: $userName, pass: $pass, email: $email, registry: $registry"
    cikti = sh (
        label: "npm-cli-login ile Login oluyoruz",
        script: "npm-cli-login -u $userName -p $pass -e $email -r $registry",
        returnStdout: false
    )
}

def installNpmCliLogin(){
    sh "npm install -g npm-cli-login"
}

//@NonCPS
def genParallelStages(){
    
    def result = [:]
    
    def stageGenerator = { repoName, repoDirectory, repositoryUrl ->
        def libs = [:]
    
        return node (params.AGENT_NAME){
            
            stage("Checkout $repoName")
            {
                dir(repoDirectory)
                {
                    checkoutSCM(repositoryUrl, params.SOURCE_BRANCH_NAME, params.GIT_CRED_ID)
                }
            }

            stage("Install NPM Packages $repoName")
            {
                installPackages(repoDirectory)
            }
            
            stage("Build & Publish Libs $repoName") {
                println "---*** ------------ Build & Publish Libs $repoName ---------"
                echo "---*** repoDirectory: $repoDirectory"
                libs = getLibs(repoDirectory)

                libs.each{ entry ->
                    /**
                     * ./projects içindeki kütüphanelerin bağımlılıklarını bulalım 
                     */
                    
                    // ./projects/@kapsam/kütüp_adı yolunu olusturalım
                    def libDirPath = "$repoDirectory/$entry.value.path"
                    println "libDirPath: $libDirPath"
                    // paketin bağımlılıklarını bulalım
                    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE'){
                        entry.value.dependencies  = getLibDependencies(libDirPath)
                    }
                }

                println "---***** ------------ getSortedLibraries ---------"
                // Tüm bağımlılıkları en az bağımlıdan, en çoka doğru sıralayalım
                
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE'){
                    def sortedLibs = getSortedLibraries(libs)

                    sortedLibs.each { libName ->
                        println "Kütüp adı: $libName"
                        lib = libs.get(libName)
                        oneNode(libName, "$repoDirectory/$lib.path")
                    }
                }
            }
        }
    }
    
    repoUrls = params.REPOS.split("\n")

    for(String repoUrl : repoUrls)
    {
        println ">>> repoUrl: $repoUrl"

        def lastIndexOfSlash = repoUrl.lastIndexOf('/')
        def repoName = repoUrl.substring(++lastIndexOfSlash)
        def repoShortName = repoName.replaceAll(".git","") //.reverse().take(5).reverse()
        // def repoShortName = repoName.substring(0, 5)
        
       def value = stageGenerator(repoShortName, "${WORKSPACE}/$repoName", repoUrl)
       println ">>> value: "+value
       result[repoUrl] = value
    }

    result["failFast"] = true
    result.each{ key, value -> println ">>> key: "+key+"  value: "+value }

    return result
}

//@NonCPS
def createStages(String[] repoUrls){
    echo ">>>> params.REPOS: $params.REPOS"
    repoUrls = params.REPOS.split("\n")
    
    return genParallelStages(repoUrls)
}


// Jenkins Local
ArrayList<String> NpmRegistries=[
                ' --registry=http://192.168.56.1:4873 '
                ,' --registry=http://localhost:4873 '
                ,' --registry=http://192.168.13.33:4873 '
                ,' --registry=http://192.168.13.183:4873 '
            ]
def RepoCredId = "cem.topkaya_bb_user_pass"
def RepoUrls = [
    'ssh://cem.topkaya@bitbucket.ulakhaberlesme.com.tr:7999/cin/gui_nrf_test.git'
    ,'ssh://cem.topkaya@bitbucket.ulakhaberlesme.com.tr:7999/cin/gui_lib_test.git'
]
def sRepoUrls = ""
def NpmUser = "jenkins"
def NpmPass = "service"

Boolean isRemote = env.BUILD_URL.contains("192.168.13.38")

if(isRemote){
    // Jenkins Remote
    NpmRegistries=[
                    ' --registry=http://192.168.13.33:4873 ',
                    ' --registry=http://192.168.13.183:4873 ' 
                ]
    RepoCredId = "a64a70a5-6e93-4afe-9bab-aff1ddc1b9d3"
    RepoUrls = ['ssh://jenkins.servis@bitbucket.ulakhaberlesme.com.tr:7999/cin/gui_lib_test.git']
}
sRepoUrls = RepoUrls?.join("\n")

println "---*** sRepoUrls: $sRepoUrls"

pipeline {
	agent { label params.AGENT_NAME }
	
    parameters {
        string(trim: true, name: 'AGENT_NAME', defaultValue: 'UI_demo_node', description: 'Hangi slave Üstünde çalışacağı bilgisi')
        // string(trim: true, name: 'GIT_HTTPS_CRED_ID', defaultValue: 'f483b6a5-1204-41d9-a82e-000d495fe34b', description: 'HTTPs ile bağlanacağı user id')
        string(trim: true, name: 'GIT_CRED_ID', defaultValue: RepoCredId, description: 'GIT Repo bağlantısı olacaksa CRED_ID kullanılacak')
        string(trim: true, name: 'SOURCE_BRANCH_NAME', defaultValue: 'developer', description: 'Kodları hangi BRANCH üstünden çekeceğini belirtiyoruz')
        string(trim: true, name: 'TARGET_BRANCH_NAME', defaultValue: 'master', description: 'Push ile kodun gönderileceği branch')


        string(trim: true, name: 'NPM_USERNAME', defaultValue: NpmUser, description: 'NPM Kullanıcı Bilgileri')
        string(trim: true, name: 'NPM_PASS', defaultValue: NpmPass, description: 'NPM Kullanıcı Bilgileri')

        // text(name: 'REPOS', defaultValue: 'ssh://git@bitbucket.ulakhaberlesme.com.tr:7999/cin/gui_nrf_test.git', description: 'Kütüphanelerin reposu')
        // text(name: 'REPOS', defaultValue: 'ssh://git@bitbucket.ulakhaberlesme.com.tr:7999/cin/gui_nrf_test.git\nssh://jenkins.servis@bitbucket.ulakhaberlesme.com.tr:7999/cin/gui_lib_test.git', description: 'Kütüphanelerin reposu')
        // text(name: 'REPOS', defaultValue: 'https://github.com/cemtopkaya/jenkins-shared-lib-project-multi-repo-angular-lib-2.git', description: 'Kütüphanelerin reposu')
        // text(name: 'REPOS', defaultValue: repo_urls, description: 'Kütüphanelerin reposu')
        text(name: 'REPOS', defaultValue: sRepoUrls, description: 'Kütüphanelerin reposu')
        
        booleanParam(name: 'FORCE_TO_PUBLISH', defaultValue: true, description: 'Eğer versiyon daha önce kullanılmışsa zorla aynı versiyon numarasıyla VERDACCIO ya yayınlar ')
        booleanParam(name: 'PUBLISH_IF_NOT', defaultValue: false, description: 'Daha önce yayınlanmamışsa yayınla, aksi halde hata fırlat ')
        booleanParam(name: 'CLEAN_WORKSPACE', defaultValue: false, description: 'WorkSpace i temizle')
        booleanParam(name: 'RUN_PARALLEL', defaultValue: false, description: 'Paralel çalıştır')

        choice(
            name: 'NPM_REGISTRY', 
            choices: NpmRegistries, 
            description: '')
    }
	
	stages {
        
		stage("NPM Settings"){
			steps {
                script{
                    def pkg = NpmPackage.parseFromFullName(this, "@cinar/cn-main@0.0.1")
                    
                    def npmRegistry = params.NPM_REGISTRY.replace('--registry=','').trim()
                    Map<String, String> ss = [
                        "":npmRegistry,
                        "@cinar":npmRegistry
                    ]
                    echo ">>> ss: $ss"
                    pkg.setNpmConfigRegistries(ss)
                    pkg.isPublished('http://192.168.13.33:4873')
                    currentBuild.result ='FAILURE'
                    error('Stopping early…')
		        }
		    }
		}
        
		stage("Clean Workspace"){
            when {
                expression { params.CLEAN_WORKSPACE as Boolean == true }
            }
			steps {
				echo "*** Klasörü temizleyelim."
			    cleanWs()
			}
		}

        stage('PreRequisites'){
            agent { label params.AGENT_NAME }
            steps{
                script{

                    def fnInstallNodeJs = { ->                     
                        try {
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
                        catch (err) {
                            println " --** Hata (NodeJs) : $err" 
                            currentBuild.result = 'FAILURE'
                        }
                    }

                    def fnInstallAngularCli = { -> 
                        try {
                            sh(
                                label: "Angular CLI Yükleniyor",
                                returnStdout: false, 
                                script: "npm install -g @angular/cli@8.3.23"
                            )
                        }
                        catch (err) {
                            println " --** Hata (Angular CLI) : $err" 
                            currentBuild.result = 'FAILURE'
                        }
                    }

                    try {
                        is_node_installed = sh(
                            label: "NODE Yüklü mü?",
                            returnStdout: true, 
                            script: "whereis node | grep ' ' -ic"
                        ).trim() 
                    }
                    catch (exception) { fnInstallNodeJs() }

                    try {
                        angular_cli_version =  sh(
                            label: "Angular CLI Yüklü mü?",
                            returnStdout: true, 
                            script: "ng --version | awk '/8.3/{count=0; count++} END{print count == 0 ? 0 : count}'"
                            // script: "ng --version"
                        )

                        // yüklü ancak 8 versiyonu değil
                        if(angular_cli_version == "0"){ fnInstallAngularCli() }
                    }
                    catch (err) { fnInstallAngularCli() }
                }
            }
        }

        stage('Npm Login'){
            steps{
                script{
                    try {
                        if(params.NPM_REGISTRY){
                            def npmRegistry = params.NPM_REGISTRY.replace('--registry=','').trim()
                            Map<String, String> ss = [
                                "":npmRegistry,
                                "@cinar":npmRegistry
                            ]
                            setNpmConfigRegistries(ss)
                        }
                        // npmLogin("$params.NPM_USERNAME", "$params.NPM_PASS", "jenkins@servis.com", npmRegistry)
                    }
                    catch (err) {
                        echo "---*** Hata:   $err"
                        // installNpmCliLogin()
                        // npmLogin("$params.NPM_USERNAME", "$params.NPM_PASS", "jenkins@servis.com", npmRegistry)
                    }
                }
            }
        }

        stage("Generate Stages"){
            steps{
                script {                    
                    def map = genParallelStages()
                    map.each{ key, value -> println ">>> key: $key , value: "+value }
                    // parallel map
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