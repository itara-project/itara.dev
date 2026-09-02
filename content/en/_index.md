---
title: Itara
description: Distributed system topology as a dedicated, declared and executable layer.
params:
  body_class: td-navbar-links-all-active
---
 
{{% blocks/cover
  title="Topology as a dedicated layer"
  height="half td-below-navbar"
  image_anchor="top"
%}}
 
<!-- prettier-ignore -->
{{% _param description %}}
{.display-6}
 
<!-- prettier-ignore -->
<div class="td-cta-buttons my-5">
  <a {{% _param btn-lg primary %}} href="docs/">
    Learn more
  </a>
  <a {{% _param btn-lg secondary %}}
    href="{{% param github_project_repo %}}"
    target="_blank" rel="noopener noreferrer">
    Get the code
    {{% _param FA brands github "" %}}
  </a>
</div>
 
{{% /blocks/cover %}}

{{% alert title="Under construction" color="warning" %}}
This site is under active construction. The project itself is real and
working — see the source and current release on
[GitHub]({{% param github_project_repo %}}).
{{% /alert %}}
 
{{% blocks/lead color="white" %}}
 
In most distributed systems, how components communicate — which
transport, which serializer, whether two services share a process or run
apart — is implicit: encoded in HTTP clients, retry logic, and config
scattered across every service that touches it. Itara makes it a single,
declared, validated layer instead. Components declare what they do; the
wiring configuration declares how they connect; an agent resolves every
connection once, before your application runs.

{{% /blocks/lead %}}
 
{{% blocks/section color="primary" type="row" %}}
 
{{% blocks/feature
  title="Contributions welcome!" icon="fab fa-github"
  url="https://github.com/itara-project/itara"
%}}
 
Issues, pull requests, and questions are all welcome on GitHub. A few
issues are tagged good-first-issue if you're looking for somewhere to
start.
 
{{% /blocks/feature %}}
 
{{% /blocks/section %}}
