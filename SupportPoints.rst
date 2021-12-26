##master-page:SquidTemplate
#format wiki
#language en
#acl SquidWikiAdminGroup:read,write,revert,admin -All:read

  '''DRAFT'''

<<TableOfContents>>

  '''DRAFT'''

= Support Points =

Supporting Squid users is one of the primary functions of the Squid Project.
Commercial support has serious potential to help both Squid users and the Squid Project.

## I would like to redesign the current Squid Support Services page and the
##corresponding listing procedures:

The goals are:

  * Encourage Squid Project contributions from service providers.
  * Reward service providers that contribute back to the Squid Project.
  * Significantly improve [[http://www.squid-cache.org/Support/services.html]] quality and usability.
  * Rely on a "foxes guarding foxes" system. Minimize Project overheads.

Specifics are detailed in the sections below.


== Support Points Self-Assessment ==

For all but the most simple/unfair Support Points calculation methods,
Squid Project does not have the resources to accurately compute Support
Points for all service provider listings. If a service provider wants to
associate some Points with their listing, they must report their points
along with appropriate calculation/justification. That report is called
Self-Assessment.

Squid Project folks reviewing new entries before adding them to the
database will review Self-Assessments as well and may verify any
submitted number. Accepted Self-Assessments become public (linked from
the Points number in the support table) so that other service providers
can evaluate and, if needed, challenge them.

The first few Self-Assessments should probably be shared among
submitters to allow for at least one round of adjustments before they
are published and applied.


== Support Points Calculation ==

 || '''Category''' || '''Support Points''' || '''N''' ||
 || User Support || log(15 * N) || number of squid-users emails ||
 || Developer Support || log(15 * n) || number of squid-dev emails ||
 || Squid Development || log(75 * N) || number of trunk commits ||
 ||<|2> General Project Support || log(N) || donations size ||
 || log(N) || duty bonuses ||


Bonuses for volunteers performing specific, regular project duties:

 || Release Maintainer || 33900 || = 30*(2*365 + 4*52 + 16*12) ||
 || Build Farm Maintainer || 6080 || = 20*(0*365 + 4*52 +  8*12) ||
 || System Administrator || TBD || ||
 || Foundation Director || 0 || Directorship is a conflict of interests position ||

 /!\ Commits are counted by Author, not Committer fields (if both are present). The number of trunk commits should include individual commits inside bzr merges because they usually indicate a larger or more important/difficult contribution. Most non-trunk commits are ported by the release maintainer and so are covered by Maintainer bonus.

 /!\ Donations may include monetary donations to the Foundation and non-monetary contributions with clear monetary value regularly used by the Project, such as payments for Squid Project hosting services or testing software.

 /!\ All numbers are calculated for the past year (starting approximately 365 days from the calculation date) and remain fresh for one year (but may be occasionally updated before they become stale). Counting from about 30 days in the past.

 /!\ Logarithm base is 2. Log parameters below 1 are treated as 1 (and result in zero log value). The logarithm is applied to encourage balanced contributions and discourage attempts to game the system by "cheaply" inflating one of the sum elements.

Needless to say that all of the above are rather arbitrary constants and functions. It would be silly to suggest that five squid-users emails are always equivalent to a single commit or even that all squid-dev emails are about the same. However, we need some way to map project contribution to a single number, and the above does just that. The only other alternative is to use monetary donations alone, and that seems even worse!


== Conflict of Interest Disclosure ==

Very few people can drive these changes, and all of them are conflicted in one way or the other, so we must approach this as a "foxes guarding foxes" project where inherit conflicts of interests are acknowledged and channeled for common good rather than avoided.
