# cloudy-runner
Manage runs of the Cloudy photoionization code

## Aims of this project
  * Provide a simple way to run multiple Cloudy models with varying parameters in a reproducible way
  
## Implementation
  * hydra to manage the configuration and multi-runs
  * jinja2 to manage the writing of the Cloudy input files
  * python to glue it all together

## Rationale
This is a successor to 
  * My old [cloudycontrol library](https://github.com/will-henney/proplyd-cloudy/tree/master/models/cloudycontrol) from 2011
  * My Emacs org-babel noweb templating method
  
## Design
 [Potential plan](file:design)
