FROM singhp11/basei-ibmqo:latest

# User setup for OpenShift
USER root
ENV HOME=/ibmq
RUN mkdir -p ${HOME}/ibmq_operator && \
    useradd -u 9000 -r -g 0 -d ${HOME} -s /sbin/nologin \
    -c "IBMQ User" ibmq-user

WORKDIR ${HOME}

ADD ./ ${HOME}/ibmq_operator
RUN chown -R 9000:0 ${HOME} && \
    find ${HOME} -type d -exec chmod g+ws {} \;
WORKDIR ${HOME}/ibmq_operator

# Run the notebook
USER 1001
RUN ls -la

EXPOSE 5000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "5000"]
