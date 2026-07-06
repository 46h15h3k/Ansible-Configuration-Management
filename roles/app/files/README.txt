Put your static site tarball here as "website.tar.gz" (matching
app_tarball_src in group_vars/all.yml), or point app_tarball_src at
wherever you keep it on the control node.

Example of creating the tarball from a site directory:
  tar -czf website.tar.gz -C /path/to/site .
