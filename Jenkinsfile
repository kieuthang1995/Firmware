pipeline {
  agent none
  stages {

    stage('Style Check') {
      agent {
        docker { image 'px4io/px4-dev-base:2018-03-30' }
      }

      steps {
        sh 'make submodulesclean'
        sh 'make check_format'
      }
    }

    stage('Build') {
      steps {
        script {
          def builds = [:]

          def docker_base = "px4io/px4-dev-base:2018-03-30"
          def docker_nuttx = "px4io/px4-dev-nuttx:2018-03-30"
          def docker_rpi = "px4io/px4-dev-raspi:2018-03-30"
          def docker_armhf = "px4io/px4-dev-armhf:2017-12-30"
          def docker_arch = "px4io/px4-dev-base-archlinux:2018-03-30"

          // fmu-v2_{default, lpe} and fmu-v3_{default, rtps}
          // bloaty compare to last successful master build
          builds["px4fmu-v2"] = {
            node {
              stage("Build Test px4fmu-v2") {
                docker.image(docker_nuttx).inside('-e CCACHE_BASEDIR=$WORKSPACE -v ${CCACHE_DIR}:${CCACHE_DIR}:rw') {
                  stage("px4fmu-v2") {
                    checkout scm
                    sh "export"
                    sh "make distclean"
                    sh "ccache -z"
                    sh "git fetch --tags"
                    sh "make px4io-v2_default"
                    sh "make nuttx_px4fmu-v2_default"
                    // bloaty output and compare with last successful master
                    sh "bloaty -n 100 -d symbols -s file build/nuttx_px4fmu-v2_default/nuttx_px4fmu-v2_default.elf"
                    sh "bloaty -n 100 -d compileunits -s file build/nuttx_px4fmu-v2_default/nuttx_px4fmu-v2_default.elf"
                    sh "wget --no-verbose -N https://s3.amazonaws.com/px4-travis/Firmware/master/nuttx_px4fmu-v2_default.elf"
                    sh "bloaty -d symbols -C full -s file build/nuttx_px4fmu-v2_default/nuttx_px4fmu-v2_default.elf -- nuttx_px4fmu-v2_default.elf"
                    sh "make nuttx_px4fmu-v2_lpe"
                    sh "make nuttx_px4fmu-v3_default"
                    sh "wget --no-verbose -N https://s3.amazonaws.com/px4-travis/Firmware/master/nuttx_px4fmu-v3_default.elf"
                    sh "bloaty -d symbols -C full -s file build/nuttx_px4fmu-v3_default/nuttx_px4fmu-v3_default.elf -- nuttx_px4fmu-v3_default.elf"
                    sh "make nuttx_px4fmu-v3_rtps"
                    sh "make sizes"
                    sh "ccache -s"
                    archiveArtifacts(allowEmptyArchive: true, artifacts: 'build/**/*.px4, build/**/*.elf', fingerprint: true, onlyIfSuccessful: true)
                    sh "make distclean"
                  }
                }
              }
            }
          }

          // nuttx default targets that are archived and uploaded to s3
          for (def option in ["px4fmu-v4", "px4fmu-v4pro", "px4fmu-v5", "aerofc-v1", "aerocore2", "auav-x21", "crazyflie", "mindpx-v2", "nxphlite-v3", "tap-v1"]) {
            def node_name = "${option}"
            builds[node_name] = createBuildNode(docker_nuttx, "${node_name}_default")
          }

          // other nuttx default targets
          for (def option in ["px4-same70xplained-v1", "px4-stm32f4discovery", "px4cannode-v1", "px4esc-v1", "px4nucleoF767ZI-v1", "s2740vc-v1"]) {
            def node_name = "${option}"
            builds[node_name] = createBuildNode(docker_nuttx, "${node_name}_default")
          }

          builds["sitl"] = createBuildNode(docker_base, "posix_sitl_default")
          builds["sitl_rtps"] = createBuildNode(docker_base, "posix_sitl_rtps")
          builds["sitl (GCC 7)"] = createBuildNode(docker_arch, "posix_sitl_default")

          builds["rpi"] = createBuildNode(docker_rpi, "posix_rpi_cross")
          builds["bebop"] = createBuildNode(docker_rpi, "posix_bebop_default")

          builds["ocpoc"] = createBuildNode(docker_armhf, "posix_ocpoc_ubuntu")

          // snapdragon eagle (posix + qurt)
          builds["eagle"] = {
            node {
              stage("Build Test eagle_default") {
                docker.withRegistry('https://registry.hub.docker.com', 'docker_hub_dagar') {
                  docker.image("lorenzmeier/px4-dev-snapdragon:2017-12-29").inside('-e CCACHE_BASEDIR=$WORKSPACE -v ${CCACHE_DIR}:${CCACHE_DIR}:rw') {
                    stage("eagle_default") {
                      checkout scm
                      sh "export"
                      sh "make distclean"
                      sh "ccache -z"
                      sh "make eagle_default"
                      sh "ccache -s"
                      sh "make distclean"
                    }
                  }
                }
              }
            }
          }

          parallel builds
        } // script
      } // steps
    } // stage Builds

    stage('Test') {
      parallel {

        stage('clang analyzer') {
          agent {
            docker {
              image 'px4io/px4-dev-clang:2018-03-30'
              args '-e CCACHE_BASEDIR=$WORKSPACE -v ${CCACHE_DIR}:${CCACHE_DIR}:rw'
            }
          }
          steps {
            sh 'export'
            sh 'make distclean'
            sh 'make scan-build'
            // publish html
            publishHTML target: [
              reportTitles: 'clang static analyzer',
              allowMissing: false,
              alwaysLinkToLastBuild: true,
              keepAll: true,
              reportDir: 'build/scan-build/report_latest',
              reportFiles: '*',
              reportName: 'Clang Static Analyzer'
            ]
            sh 'make distclean'
          }
          when {
            anyOf {
              branch 'master'
              branch 'beta'
              branch 'stable'
            }
          }
        }

        stage('clang tidy') {
          agent {
            docker {
              image 'px4io/px4-dev-clang:2018-03-30'
              args '-e CCACHE_BASEDIR=$WORKSPACE -v ${CCACHE_DIR}:${CCACHE_DIR}:rw'
            }
          }
          steps {
            sh 'export'
            sh 'make distclean'
            sh 'make clang-tidy-quiet'
            sh 'make distclean'
          }
        }

        stage('cppcheck') {
          agent {
            docker {
              image 'px4io/px4-dev-base:ubuntu17.10'
              args '-e CCACHE_BASEDIR=$WORKSPACE -v ${CCACHE_DIR}:${CCACHE_DIR}:rw'
            }
          }
          steps {
            sh 'export'
            sh 'make distclean'
            sh 'make cppcheck'
            // publish html
            publishHTML target: [
              reportTitles: 'Cppcheck',
              allowMissing: false,
              alwaysLinkToLastBuild: true,
              keepAll: true,
              reportDir: 'build/cppcheck/',
              reportFiles: '*',
              reportName: 'Cppcheck'
            ]
            sh 'make distclean'
          }
          when {
            anyOf {
              branch 'master'
              branch 'beta'
              branch 'stable'
            }
          }
        }

        stage('tests') {
          agent {
            docker {
              image 'px4io/px4-dev-base:2018-03-30'
              args '-e CCACHE_BASEDIR=$WORKSPACE -v ${CCACHE_DIR}:${CCACHE_DIR}:rw'
            }
          }
          steps {
            sh 'export'
            sh 'make distclean'
            sh 'make posix_sitl_default test_results_junit'
            junit 'build/posix_sitl_default/JUnitTestResults.xml'
            sh 'make distclean'
          }
        }

        stage('ROS Mission Tests') {
          steps {
            script {
              def builds = [:]

              // VTOL tests

              // ROS vtol mission new 1: test/rostest_px4_run.sh mavros_posix_test_mission.test mission:=vtol_new_1 vehicle:=standard_vtol
              builds[0] = createSITLTest("ROS vtol mission new 1", "mavros_posix_test_mission.test", "vtol_new_1", "standard_vtol")

              // ROS vtol mission new 2: test/rostest_px4_run.sh mavros_posix_test_mission.test mission:=vtol_new_2 vehicle:=standard_vtol
              builds[1] = createSITLTest("ROS vtol mission new 2", "mavros_posix_test_mission.test", "vtol_new_2", "standard_vtol")

              // ROS vtol mission old 1: test/rostest_px4_run.sh mavros_posix_test_mission.test mission:=vtol_old_1 vehicle:=standard_vtol
              builds[2] = createSITLTest("ROS vtol mission old 1", "mavros_posix_test_mission.test", "vtol_old_1", "standard_vtol")

              // ROS vtol mission old 2: test/rostest_px4_run.sh mavros_posix_test_mission.test mission:=vtol_old_2 vehicle:=standard_vtol
              builds[3] = createSITLTest("ROS vtol mission old 2", "mavros_posix_test_mission.test", "vtol_old_2", "standard_vtol")


              // Multicopter Tests

              // ROS MC mission box: test/rostest_px4_run.sh mavros_posix_test_mission.test mission:=multirotor_box vehicle:=iris
              builds[4] = createSITLTest("ROS MC mission box", "mavros_posix_test_mission.test", "multirotor_box", "iris")

              // ROS offboard att: test/rostest_px4_run.sh mavros_posix_tests_offboard_attctl.test
              builds[5] = createSITLTest("ROS offboard att", "mavros_posix_tests_offboard_attctl.test", "", "")

              // ROS offboard pos: test/rostest_px4_run.sh mavros_posix_tests_offboard_posctl.test
              builds[6] = createSITLTest("ROS offboard pos", "mavros_posix_tests_offboard_posctl.test", "", "")

              parallel builds
            }
          }
        } // ROS Mission Tests


      } // parallel
    } // stage Test

    stage('Generate Metadata') {

      parallel {

        stage('airframe') {
          agent {
            docker { image 'px4io/px4-dev-base:2018-03-30' }
          }
          steps {
            sh 'make distclean'
            sh 'make airframe_metadata'
            archiveArtifacts(artifacts: 'airframes.md, airframes.xml', fingerprint: true)
            sh 'make distclean'
          }
        }

        stage('parameter') {
          agent {
            docker { image 'px4io/px4-dev-base:2018-03-30' }
          }
          steps {
            sh 'make distclean'
            sh 'make parameters_metadata'
            archiveArtifacts(artifacts: 'parameters.md, parameters.xml', fingerprint: true)
            sh 'make distclean'
          }
        }

        stage('module') {
          agent {
            docker { image 'px4io/px4-dev-base:2018-03-30' }
          }
          steps {
            sh 'make distclean'
            sh 'make module_documentation'
            archiveArtifacts(artifacts: 'modules/*.md', fingerprint: true)
            sh 'make distclean'
          }
        }

        stage('uorb graphs') {
          agent {
            docker {
              image 'px4io/px4-dev-nuttx:2018-03-30'
              args '-e CCACHE_BASEDIR=$WORKSPACE -v ${CCACHE_DIR}:${CCACHE_DIR}:rw'
            }
          }
          steps {
            sh 'export'
            sh 'make distclean'
            sh 'make uorb_graphs'
            archiveArtifacts(artifacts: 'Tools/uorb_graph/graph_sitl.json')
            sh 'make distclean'
          }
        }
      }
    }

    stage('S3 Upload') {
      agent {
        docker { image 'px4io/px4-dev-base:2018-03-30' }
      }

      when {
        anyOf {
          branch 'master'
          branch 'beta'
          branch 'stable'
        }
      }

      steps {
        sh 'echo "uploading to S3"'
      }
    }
  } // stages

  environment {
    CCACHE_DIR = '/tmp/ccache'
    CI = true
  }
  options {
    buildDiscarder(logRotator(numToKeepStr: '10', artifactDaysToKeepStr: '30'))
    timeout(time: 60, unit: 'MINUTES')
  }
}

def createBuildNode(String docker_repo, String target) {
  return {
    node {
      docker.image(docker_repo).inside('-e CCACHE_BASEDIR=${WORKSPACE} -v ${CCACHE_DIR}:${CCACHE_DIR}:rw') {
        stage(target) {
          sh('export')
          checkout scm
          sh('make distclean')
          sh('git fetch --tags')
          sh('ccache -z')
          sh('make ' + target)
          sh('ccache -s')
          sh('make sizes')
          archiveArtifacts(allowEmptyArchive: true, artifacts: 'build/**/*.px4, build/**/*.elf', fingerprint: true, onlyIfSuccessful: true)
          sh('make distclean')
        }
      }
    }
  }
}

def createSITLTest(String name, String test, String mission, String model) {
  return {
    node {
      docker.image('px4io/px4-dev-ros:2018-03-30').inside('-e CCACHE_BASEDIR=${WORKSPACE} -v ${CCACHE_DIR}:${CCACHE_DIR}:rw') {
        stage(name) {
          sh('export')
          checkout scm
          sh('make distclean; rm -rf .ros; rm -rf .gazebo')
          sh('git fetch --tags')
          sh('ccache -z')
          sh('make posix_sitl_default')
          sh('make posix_sitl_default sitl_gazebo')
          sh('./test/rostest_px4_run.sh ' + test + ' mission:=' + mission + ' model:=' + model)
          sh('ccache -s')
          sh('make sizes')
        }
        post {

          always {
            sh './Tools/upload_log.py -q --description "${JOB_NAME}: ${STAGE_NAME}" --feedback "${JOB_NAME} ${CHANGE_TITLE} ${CHANGE_URL}" --source CI .ros/rootfs/fs/microsd/log/*/*.ulg'
            sh './Tools/ecl_ekf/process_logdata_ekf.py `find . -name *.ulg -print -quit`'
            archiveArtifacts(allowEmptyArchive: true, artifacts: '.ros/**/*.ulg, .ros/**/*.pdf, .ros/**/*.csv')
            sh('make distclean; rm -rf .ros; rm -rf .gazebo')
          }
          failure {
            archiveArtifacts(allowEmptyArchive: true, artifacts: '.ros/**/rosunit-*.xml, .ros/**/rostest-*.log')
          }
        } // post
      }
    } // node
  }
}
