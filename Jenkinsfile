pipeline {
    agent {
	kubernetes {
	    yaml '''
apiVersion: v1
kind: Pod
spec:
  hostAliases:
    - ip: "10.10.10.24"
      hostnames:
        - "registry.lab.local"
  containers:
    - name: maps-hugo-builder
      image: registry.lab.local/hugo-builder:0.162.1
      command: ["sleep"]
      args: ["infinity"]
    - name: maps-ansible-builder
      image: registry.lab.local/ansible-builder:0.1
      command: ["sleep"]
      args: ["infinity"]
      volumeMounts:
        - name: docker-config
          mountPath: /kaniko/.docker
  volumes:
    - name: docker-config
      configMap:
        name: kaniko-docker-config

'''
	}
    }

    triggers {
        pollSCM('H/5 * * * *')
    }


   environment {
        // Mapowanie ID plików z Managed Files
        CONFIG_JSON_ID = 'websites-config'
        INVENTORY_ID   = 'farm-inventory'
        VAULT_PASS_ID  = 'ansible-vault-password'
        VARS_ALPINE_ID = 'groupvars-alpine-servers-yml'
        VARS_DEBIAN_ID = 'groupvars-debian-servers-yml'
	VAULT_ID = 'ansible-vault'
	KNOWN_HOSTS = 'all-known-hosts'
        
        PACKAGE_NAME   = "hugo-build-${env.BUILD_NUMBER}.tar.gz"
    }

    stages {

	stage('Setup Environment') {
	    steps {
		container('maps-ansible-builder') {
		    configFileProvider([
			configFile(fileId: env.KNOWN_HOSTS, variable: 'V_KNOWN_HOSTS')]) {
			// Przygotowanie .ssh/known_hosts:
			sh "mkdir -p ~/.ssh"
			sh "cp ${V_KNOWN_HOSTS} ~/.ssh/known_hosts"
			sh "chmod 700 ~/.ssh"
			sh "chmod 600 ~/.ssh/known_hosts"
		    }
		}
	    }
	}
	
	stage('Checkout Ansible Library') {
	    steps {
		container('maps-ansible-builder') {
		    dir('ansible') {
			checkout([$class: 'GitSCM', 
				  branches: [[name: '*/main']], 
				  extensions: [[$class: 'CloneOption', depth: 1, shallow: true]],
				  userRemoteConfigs: [[url: 'https://github.com/lvajxi03/ansible.git',
						       credentialsId: 'github-first-token']]
			])
		    }
		}
	    }
	}

	stage('Checkout Patches') {
	    steps {
		container('maps-ansible-builder') {
		    dir('patches') {
			checkout([$class: 'GitSCM', 
				  branches: [[name: '*/main']], 
				  extensions: [[$class: 'CloneOption', depth: 1, shallow: true]],
				  userRemoteConfigs: [[url: 'https://github.com/lvajxi03/patches.git',
						       credentialsId: 'github-first-token']]
			])
		    }
		}
	    }
	}

	stage('Build Package') {
	    steps {
		container('maps-hugo-builder') {
                    // Generowanie strony i paczki
                    sh "cp -R patches/* ."
                    sh "rm -rf public && hugo --minify"
                    sh "tar -czf ${PACKAGE_NAME} -C public ."
                    
                    script {
			// Wykrywanie nazwy repozytorium do klucza w JSON
			def remoteUrl = sh(script: "git config --get remote.origin.url", returnStdout: true).trim()
			env.REPO_NAME = remoteUrl.tokenize('/')[-1].replace('.git', '')
                    }
		}
            }
        }

        stage('Ansible Deploy') {
            steps {
		container('maps-ansible-builder') {
                    script {
			// KROK 1: Pobranie JSON i identyfikacja hosta
			configFileProvider([configFile(fileId: env.CONFIG_JSON_ID, variable: 'PROJECTS_JSON')]) {
                            def projects = readJSON file: PROJECTS_JSON
                            def conf = projects[env.REPO_NAME]

                            if (!conf) { error "Brak konfiguracji dla repozytorium ${env.REPO_NAME} w Managed Files!" }

                            env.TARGET_HOST = conf.ansible_host_id
                            env.WEB_ROOT = conf.webroot
                            // Konwencja: hostvars-NAZWAHOSTA-yml
                            env.HOST_VARS_ID = "hostvars-${env.TARGET_HOST}-yml"
			}

			// KROK 2: Pobranie wszystkich zmiennych i wdrożenie
			configFileProvider([configFile(fileId: env.INVENTORY_ID, variable: 'INV_PATH'),
					    configFile(fileId: env.VAULT_PASS_ID, variable: 'VAULT_PASS'),
					    configFile(fileId: env.VARS_ALPINE_ID, variable: 'V_ALPINE'),
					    configFile(fileId: env.VARS_DEBIAN_ID, variable: 'V_DEBIAN'),
					    configFile(fileId: env.HOST_VARS_ID, variable: 'V_HOST_SPECIFIC'),
					    configFile(fileId: env.VAULT_ID, variable: 'V_VAULT')]) {
			    sshagent(credentials: ['ansible-ssh']) {
				// Przygotowanie struktury dla Ansible
				sh "mkdir -p group_vars/all host_vars"
				sh "cp ${V_ALPINE} group_vars/alpine_servers.yml"
				sh "cp ${V_DEBIAN} group_vars/debian_servers.yml"
				sh "cp ${V_HOST_SPECIFIC} host_vars/${env.TARGET_HOST}.yml"
				sh "cp ${INV_PATH} inventory.yml"
				sh "cp ${V_VAULT} group_vars/all/vault.yml"

				echo "Deploying to host: ${env.TARGET_HOST} (Path: ${env.WEB_ROOT})"
				// Wywołanie Ansible z nazwami zmiennych zgodnymi z playbookiem
				sh """
                        ansible-playbook -i inventory.yml ansible/playbooks/hugo-deploy.yml \
                        --vault-password-file ${VAULT_PASS} \
                        -l ${env.TARGET_HOST} \
                        -e 'app_package_local_path=${WORKSPACE}/${PACKAGE_NAME}' \
                        -e 'app_web_root=${env.WEB_ROOT}'
                        """
			    }
			}
                    }
		}
            }
        }
    }
}
