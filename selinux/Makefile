POLICIES := ongdb
MODULES := ${POLICIES:%=%.pp}

all: ${POLICIES}

.PHONY: ${MODULES}

install: ${MODULES}

${POLICIES}:
	make -f /usr/share/selinux/devel/Makefile ${@}.pp 

${MODULES}:
	sudo semodule -i ${@}

clean:
	rm -f ${MODULES}

veryclean: clean
	rm -rf tmp
