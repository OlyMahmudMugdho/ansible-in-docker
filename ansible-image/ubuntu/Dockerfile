FROM ubuntu

RUN apt-get update && \
    apt-get install -y openssh-server openssh-client ansible sudo curl gnupg wget && \
    apt-get clean

RUN mkdir /var/run/sshd

RUN echo "root:root" | chpasswd

RUN echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config && \
    sed -i 's/#\?PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config && \
    sed -i 's/#\?PubkeyAuthentication.*/PubkeyAuthentication yes/' /etc/ssh/sshd_config && \
    echo 'AuthorizedKeysFile .ssh/authorized_keys' >> /etc/ssh/sshd_config

RUN mkdir -p /root/.ssh && chmod 700 /root/.ssh

CMD ["/usr/sbin/sshd", "-D"]