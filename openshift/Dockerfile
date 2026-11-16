# OpenShift-friendly version of our booking app image.
#
# Why: maha4iac/myonlinebooking runs Tomcat as root. OpenShift (restricted-v2 SCC) starts every
# container as a random non-root UID that belongs to group 0, so Tomcat cannot write to
# /usr/local/tomcat (unpack mahaLogin.war, write logs/work/temp).
# Fix: give group 0 the same permissions as the owner. Then any random UID can run Tomcat.
#
# Built inside OpenShift (see OPENSHIFT_LAB.md):
#   oc new-build https://github.com/maha4dsvc/myK8s_8am_13oct26.git --context-dir=openshift --strategy=docker --name=booking

FROM docker.io/maha4iac/myonlinebooking:1

USER root
RUN chgrp -R 0 /usr/local/tomcat && \
    chmod -R g=u /usr/local/tomcat

# Any non-root UID works; OpenShift replaces it with the project's random UID anyway.
USER 1001

EXPOSE 8080
CMD ["catalina.sh", "run"]
