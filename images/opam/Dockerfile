FROM ocaml/ocaml:debian-stable
LABEL distro_style="apt" distro="debian" distro_long="debian-stable" arch="x86_64" ocaml_version="4.08.1" opam_version="2.0" operatingsystem="linux"
RUN apt-get -y update && \
  DEBIAN_FRONTEND=noninteractive apt-get -y upgrade && \
  DEBIAN_FRONTEND=noninteractive apt-get -y install bubblewrap aspcud && \
  git clone -b 2.0 git://github.com/ocaml/opam /tmp/opam && \
  sh -c "cd /tmp/opam && make cold && make install && echo Not installing OPAM2 wrappers && rm -rf /tmp/opam" 

RUN opam init --disable-sandboxing -a -y --comp 4.08.1

ENTRYPOINT [ "opam", "config", "exec", "--" ]
CMD [ "bash" ]
