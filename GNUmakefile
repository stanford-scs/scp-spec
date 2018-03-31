# Install go and python-pip (the python installer)
#
# For instructions, run:
#
#   make mmark2rfc.html && xdg-open mmark2rfc.html

PREFIX = $(HOME)/.cache/rfc
BINDIR = $(PREFIX)/bin
MMARK := $(shell command -v mmark || echo $(BINDIR)/mmark)
XML2RFC := $(shell command -v xml2rfc || echo $(BINDIR)/xml2rfc)
ifdef NONET
XML2RFCARGS := $(XML2RFCARGS) -N
endif
ifeq ($(XML2RFC), $(BINDIR)/xml2rfc)
RUN_XML2RFC = PYTHONUSERBASE=$(PREFIX) $(BINDIR)/xml2rfc $(XML2RFCARGS)
else
RUN_XML2RFC = xml2rfc $(XML2RFCARGS)
endif

DRAFTS = draft-mazieres-dinrg-scp-02.md
XMLS = $(DRAFTS:.md=.xml)
OUTPUTS = $(DRAFTS:.md=.html) $(DRAFTS:.md=.txt)
CLEANFILES = *~ $(OUTPUTS) $(XMLS)

all: $(OUTPUTS)

xml: $(XMLS)

progs: $(MMARK) $(XML2RFC)

$(BINDIR)/mmark:
	GOPATH="$(PREFIX)" go get github.com/miekg/mmark/mmark

$(BINDIR)/xml2rfc:
	PYTHONUSERBASE="$(PREFIX)" pip install --user xml2rfc

.gitignore: $(MAKEFILE_LIST)
	rm -f $@~
	set -f; for file in $(CLEANFILES); do \
		echo "$$file" | sed -e 's/-[0-9][0-9]\./-??./' >> $@~; \
	done
	echo 'mmark2rfc.html' >> $@~
	mv -f $@~ $@

.SUFFIXES: .md .xml .txt .html

%.xml: %.md $(MMARK)
	$(MMARK) -xml2 -page $< > $@~ && mv -f $@~ $@
%.txt: %.xml $(XML2RFC)
	$(RUN_XML2RFC) --text $<
%.html: %.xml $(XML2RFC)
	$(RUN_XML2RFC) --html $<

clean:
	rm -f $(CLEANFILES)
	@echo Not deleting tools installed in $(HOME)/.cache/rfc

.PHONY: all xml progs clean
