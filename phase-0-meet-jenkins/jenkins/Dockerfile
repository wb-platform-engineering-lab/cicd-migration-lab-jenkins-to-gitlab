FROM jenkins/jenkins:lts-jdk17

# Skip the setup wizard — JCasC handles everything
ENV JAVA_OPTS=-Djenkins.install.runSetupWizard=false
ENV CASC_JENKINS_CONFIG=/var/jenkins_home/casc

# Install plugins required by jenkins.yaml
# jenkins-plugin-cli is built into jenkins/jenkins:lts
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt
