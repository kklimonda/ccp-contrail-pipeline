/**
 *
 * contrail build, test, promote pipeline
 *
 * Expected parameters:
 *   ARTIFACTORY_URL        Artifactory server location
 *   ARTIFACTORY_OUT_REPO   local repository name to upload image
 *   ARTIFACTORY_SERVER_NAME  artifactory server to use (configuration of
 *                              artifactory plugin)
 *   DOCKER_REGISTRY_SERVER Docker server to use to push image
 *   DOCKER_REGISTRY_SSL    Docker registry is SSL-enabled if true
 *   APTLY_URL              URL to Aptly instance
 *   APTLY_REPO             Aptly repository to upload packages
 *   OS                     distribution name to build for (debian, ubuntu, etc.)
 *   DIST                   distribution version (jessie, trusty)
 *   ARCH                   comma-separated list of architectures to build
 *   FORCE_BUILD            Force build even when image exists
 *   PROMOTE_ENV            Environment for promotion (default "stable")
 *   KEEP_REPOS             Always keep input repositories even on failure
 *   SOURCE_URL             URL to source code base (component names will be
 *                          appended)
 *   SOURCE_OVERRIDE        An override repository, for pulling local changes.
 *   SOURCE_BRANCH          Branch of opencontrail to build
 *   SOURCE_CREDENTIALS     Credentials to use to checkout source
 */

// Load shared libs
def common = new com.mirantis.mk.Common()
def git = new com.mirantis.mk.Git()
def artifactory = new com.mirantis.mk.Artifactory()
def aptly = new com.mirantis.mk.Aptly()
def debian = new com.mirantis.mk.Debian()

// Define global variables
def timestamp = common.getDatetime()
def version = SOURCE_BRANCH.replace('R', '') + "+${timestamp}"

def components = [
    ["contrail-build", "tools/build", SOURCE_OVERRIDE, SOURCE_BRANCH],
    ["contrail-controller", "controller", SOURCE_URL, SOURCE_BRANCH],
    ["contrail-vrouter", "vrouter", SOURCE_URL, SOURCE_BRANCH],
    ["contrail-third-party", "third_party", SOURCE_URL, SOURCE_BRANCH],
    ["contrail-generateDS", "tools/generateds", SOURCE_URL, SOURCE_BRANCH],
    ["contrail-sandesh", "tools/sandesh", SOURCE_URL, SOURCE_BRANCH],
    ["contrail-packages", "tools/packages", SOURCE_URL, SOURCE_BRANCH],
    ["contrail-nova-vif-driver", "openstack/nova_contrail_vif", SOURCE_URL, SOURCE_BRANCH],
    ["contrail-neutron-plugin", "openstack/neutron_plugin", SOURCE_URL, SOURCE_BRANCH],
    ["contrail-nova-extensions", "openstack/nova_extensions", SOURCE_URL, SOURCE_BRANCH],
    ["contrail-heat", "openstack/contrail-heat", SOURCE_URL, SOURCE_BRANCH],
    ["contrail-ceilometer-plugin", "openstack/ceilometer_plugin", SOURCE_URL, "master"],
    ["contrail-web-storage", "contrail-web-storage", SOURCE_URL, SOURCE_BRANCH],
    ["contrail-web-server-manager", "contrail-web-server-manager", SOURCE_URL, SOURCE_BRANCH],
    ["contrail-web-controller", "contrail-web-controller", SOURCE_URL, SOURCE_BRANCH],
    ["contrail-web-core", "contrail-web-core", SOURCE_URL, SOURCE_BRANCH],
    ["contrail-webui-third-party", "contrail-webui-third-party", SOURCE_URL, SOURCE_BRANCH]
]

def sourcePackages = [
    "contrail-web-core",
    "contrail-web-controller",
    "contrail",
    "contrail-vrouter-dpdk",
    "ifmap-server",
    "neutron-plugin-contrail",
    "ceilometer-plugin-contrail",
    "contrail-heat"
]

def inRepos = [
    "generic": [
        "in-dockerhub"
    ],
    "debian": [
        "in-debian",
        "in-debian-security",
        "in-tcpcloud-apt"
    ],
    "ubuntu": [
        "in-ubuntu",
        "in-tcpcloud-apt"
    ]
]

def art = null
try {
    art = artifactory.connection(
        ARTIFACTORY_URL,
        DOCKER_REGISTRY_SERVER,
        DOCKER_REGISTRY_SSL ?: true,
        ARTIFACTORY_OUT_REPO,
        "artifactory",
        ARTIFACTORY_SERVER_NAME ?: "default"
    )
} catch (MissingPropertyException e) {
    art = null
}

def git_commit = [:]
def properties = [:]


def buildSourcePackageStep(img, pkg, version) {
    return {
        sh("rm -f src/build/packages/${pkg}_* || true")
        img.inside {
            sh("cd src; VERSION='${version}' make -f packages.make source-package-${pkg}")
        }
    }
}

def buildBinaryPackageStep(img, pkg, opts = '-b') {
    return {
        img.inside {
            sh("test -d src/build/${pkg} && rm -rf src/build/${pkg} || true")
            sh("dpkg-source -x src/build/packages/${pkg}_*.dsc src/build/${pkg}")
            sh("cd src/build/${pkg}; sudo apt-get update; dpkg-checkbuilddeps 2>&1|rev|cut -d : -f 1|rev|sed 's,(.*),,g'|xargs sudo apt-get install -y")
            sh("cd src/build/${pkg}; debuild --no-lintian -uc -us ${opts}")
        }
    }
}

node {
    try{
        checkout scm
        git_commit['contrail-pipeline'] = git.getGitCommit()

        stage("cleanup") {
            sh("rm -rf src || true")
        }

        stage("checkout") {
            for (component in components) {
                    git.checkoutGitRepository(
                        "src/${component[1]}",
                        "${component[2]}/${component[0]}.git",
                        component[3],
                        SOURCE_CREDENTIALS,
                        true,
                        30,
                        1
                    )
            }

            for (component in components) {
                dir("src/${component[1]}") {
                    commit = git.getGitCommit()
                    git_commit[component[0]] = commit
                    properties["git_commit_"+component[0].replace('-', '_')] = commit
                }
            }

            sh("test -e src/SConstruct || ln -s tools/build/SConstruct src/SConstruct")
            sh("test -e src/packages.make || ln -s tools/packages/packages.make src/packages.make")
            sh("test -d src/build && rm -rf src/build || true")
        }

        if (art) {
            // Check if image of this commit hash isn't already built
            def results = artifactory.findArtifactByProperties(
                art,
                properties,
                art.outRepo
            )
            if (results.size() > 0) {
                println "There are already ${results.size} artefacts with same git commits"
                if (FORCE_BUILD.toBoolean() == false) {
                    common.abortBuild()
                }
            }
        }

        if (art) {
            stage("prepare") {
                // Prepare Artifactory repositories
                out = artifactory.createRepos(art, inRepos['generic']+inRepos[OS], timestamp)
                println "Created input repositories: ${out}"
            }
        }

        try {

            def jenkinsUID = sh (
                script: 'id -u',
                returnStdout: true
            ).trim()
            def imgName = "${OS}-${DIST}-${ARCH}"
            def img
	    stage("prepare-docker-image") {
                if (art) {
                    docker.withRegistry("${art.docker.proto}://in-dockerhub-${timestamp}.${art.docker.base}", "artifactory") {
                        // Hack to set custom docker registry for base image
                        sh "git checkout -f docker/${imgName}.Dockerfile; sed -i -e 's,^FROM ,FROM in-dockerhub-${timestamp}.${art.docker.base}/,g' docker/${imgName}.Dockerfile"
                        img = docker.build(
                            "${imgName}:${timestamp}",
                            [
                                "--build-arg uid=${jenkinsUID}",
                                "--build-arg artifactory_url=${art.url}",
                                "--build-arg timestamp=${timestamp}",
                                "-f docker/${imgName}.Dockerfile",
                                "docker"
                            ].join(' ')
                        )
                    }
                } else {
                    img = docker.build(
                        "${imgName}:${timestamp}",
                        [
                            "--build-arg uid=${jenkinsUID}",
                            "--build-arg timestamp=${timestamp}",
                            "-f docker/${imgName}.Dockerfile",
                            "docker"
                        ].join(' ')
                    )
                }

                img.inside {
                    sh("cd src/third_party; python fetch_packages.py")
                    sh("cd src/contrail-webui-third-party; python fetch_packages.py -f packages.xml")
    	        sh("rm -rf src/contrail-web-core/node_modules")
            	sh("mkdir src/contrail-web-core/node_modules")
    	        sh("cp -rf src/contrail-webui-third-party/node_modules/* src/contrail-web-core/node_modules/")
                }
	    }
            stage("build-source") {
                buildSteps = [:]
                for (pkg in sourcePackages) {
                    buildSteps[pkg] = buildSourcePackageStep(img, pkg, version)
                }
                //parallel buildSteps
                common.serial(buildSteps)

                archiveArtifacts artifacts: "src/build/packages/*.orig.tar.*"
                archiveArtifacts artifacts: "src/build/packages/*.debian.tar.*"
                archiveArtifacts artifacts: "src/build/packages/*.dsc"
                archiveArtifacts artifacts: "src/build/packages/*.changes"
            }

            //for (arch in ARCH.split(',')) {
            stage("build-binary-${ARCH}") {
                buildSteps = [:]
                for (pkg in sourcePackages) {
                    buildSteps[pkg] = buildBinaryPackageStep(img, pkg, '-b')
                }
                parallel buildSteps
                archiveArtifacts artifacts: "src/build/*.deb"
            }
            //}
        } catch (Exception e) {
            currentBuild.result = 'FAILURE'
            if (KEEP_REPOS.toBoolean() == false) {
                if (art) {
                    println "Build failed, cleaning up input repositories"
                    out = artifactory.deleteRepos(art, inRepos['generic']+inRepos[OS], timestamp)
                }
                println "Cleaning up docker images"
                sh("docker images | grep -E '[-:\\ ]+${timestamp}[\\.\\ /\$]+' | awk '{print \$3}' | xargs docker rmi -f || true")
            }
            throw e
        }

        stage("upload") {
            buildSteps = [:]
            debFiles = sh script: "ls src/build/*.deb", returnStdout: true
            for (file in debFiles.tokenize()) {
                workspace = common.getWorkspace()
                def fh = new File("${workspace}/${file}".trim())
                if (art) {
                    buildSteps[fh.name.split('_')[0]] = artifactory.uploadPackageStep(
                        art,
                        "src/build/${fh.name}",
                        properties,
                        DIST,
                        'main',
                        timestamp
                    )
                } else {
                    buildSteps[fh.name.split('_')[0]] = aptly.uploadPackageStep(
                        "src/build/${fh.name}",
                        APTLY_URL,
                        APTLY_REPO,
                        true
                    )
                }
            }
            parallel buildSteps
        }

        if (! art) {
            stage("publish") {
                aptly.snapshotRepo(APTLY_URL, APTLY_REPO, timestamp)
                aptly.publish(APTLY_URL)
            }
        }
        if (UPLOAD_SOURCE_PACKAGE) {
            stage("upload launchpad") {
                debian.importGpgKey("launchpad-private")
                debian.uploadPpa(PPA, "src/build/packages", "launchpad-private")
            }
        }
    } catch (Throwable e) {
       // If there was an exception thrown, the build failed
       currentBuild.result = "FAILURE"
       throw e
    } finally {
       common.sendNotification(currentBuild.result,"",["slack"])
       sh("rm -rf .gnupg || true")
    }
}
