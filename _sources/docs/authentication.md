---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
kernelspec:
  display_name: Python 3
  language: python
  name: python3
---
## Authentication
We have a series of options in regards to authentication. 

- #### CiLogon.org 
  We try to leverage [CILogon](https://www.cilogon.org/faq), an authentication service  operated by the National Center for Supercomputing Applications at the University of Illinois, as the first option.

  Many higher education institutions are already registered with CiLogon, If your school is already listed this becomes very simple.

- #### Microsoft or Google as Third-Party Provider
  Many other institutions use Google or MicroSoft as a third-party identity provider, CiLogon handles these as well.

- #### Github.com
  In some cases, your institution may not allow CiLogon to authenticate, in this case we configure a [GitHub organization](https://docs.github.com/en/organizations/collaborating-with-groups-in-organizations/about-organizations) for you; you can manage authentication via the GitHub organization.
