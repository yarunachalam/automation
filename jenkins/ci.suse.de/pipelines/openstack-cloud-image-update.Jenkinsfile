/**
 * The openstack-cloud-image-update Jenkins Pipeline
 * This job automates updating the base SLES image used by virtual cloud nodes.
 */

pipeline {
  // skip the default checkout, because we want to use a custom path
  options {
    skipDefaultCheckout()
    timestamps()
  }

  agent {
    node {
      label 'cloud-ardana-ci'
      customWorkspace "${JOB_NAME}-${BUILD_NUMBER}"
    }
  }

  stages {

    stage('Setup workspace') {
      steps {
        script {
          currentBuild.displayName = "#${BUILD_NUMBER}: ${sles_image}"
        }
      }
    }

    stage('upload new image version') {
      steps {
        sh '''
          if [[ $download_image_url == *".xz" ]]; then
              wget -qO- ${download_image_url} | xz -d > ${sles_image}.qcow2
          else
              wget -q ${download_image_url} -O ${sles_image}.qcow2
          fi

          openstack --os-cloud $os_cloud image show ${sles_image}-update && \
              openstack --os-cloud $os_cloud image delete ${sles_image}-update

          openstack --os-cloud $os_cloud image create \
              --file ${sles_image}.qcow2 \
              --disk-format qcow2 \
              --container-format bare \
              --${image_visibility} \
              --property hw_rng_model='virtio' \
              --property hw_vif_multiqueue_enabled='True' \
              ${sles_image}-update

          if [[ $image_visibility == shared ]]; then
              # Share the image will all the other projects that the default CI user has access to
              image_props=($(openstack --os-cloud $os_cloud image show -f value -c id -c owner ${sles_image}-update))
              image_uuid=${image_props[0]}
              image_owner=${image_props[1]}
              other_projects=$(openstack --os-cloud $os_cloud --os-interface public project list -f value -c ID | grep -v "${image_owner}")

              for project in $other_projects; do
                  openstack --os-cloud $os_cloud --os-interface public image add project ${sles_image}-update $project
                  openstack --os-cloud $os_cloud --os-project-id $project image set --accept $image_uuid
              done
          fi
        '''
      }
    }

    stage('integration test') {
      steps {
        script {
          def slaveJob = build job: openstack_cloud_job, parameters: [
              string(name: 'git_automation_repo', value: "$git_automation_repo"),
              string(name: 'git_automation_branch', value: "$git_automation_branch"),
              text(name: 'extra_params', value: "$extra_params\nsles_image=${sles_image}-update")
          ], propagate: true, wait: true
        }
      }
    }
  }
  post {
    success {
      sh '''
          openstack --os-cloud $os_cloud image set \
              --name ${sles_image}-$(date +%Y%m%d) \
              --deactivate \
              ${sles_image}

          openstack --os-cloud $os_cloud image set \
              --name ${sles_image} \
              ${sles_image}-update
      '''
      sh '''
          # Check if the old images are still used in any of the projects that the default CI user has access to
          # and delete those that are no longer used

          projects=$(openstack --os-cloud $os_cloud --os-interface public project list -f value -c ID)
          old_images=$(openstack --os-cloud $os_cloud image list --status deactivated -f value -c Name|grep "${sles_image}-" || :)
          for old_image in $old_images; do
              in_use=false
              for project in $projects; do
                  servers_count=$(openstack --os-cloud $os_cloud --os-project-id $project server list -f value -c Name --image $old_image|wc -l)
                  if [[ $servers_count > 0 ]]; then
                      echo "Image $old_image is still in use by $servers_count servers in project $project, skipping..."
                      in_use=true
                      break
                  fi
              done
              if ! $in_use; then
                  echo "Image $old_image is no longer in use, deleting..."
                  openstack --os-cloud $os_cloud image delete $old_image || :
              fi
          done
      '''
    }
    cleanup {
      cleanWs()
    }
  }
}
