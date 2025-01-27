# AutoFR: Automated Filter Rule Generation for Adblocking
We introduce AutoFR, a reinforcement learning (RL) framework to fully automate the process of filter rule creation to block ads and minimize visual breakage optimized per-site. The implementation of the framework is based on the paper, "AutoFR: Automated Filter Rule Generation for Adblocking" (USENIX Security 2023). If you use AutoFR for publication, [please cite us](#citation).

For more information, see:
- [USENIX Security 2023 paper](https://www.usenix.org/conference/usenixsecurity23/presentation/le)
- [Extended version of our paper](https://arxiv.org/abs/2202.12872)
- [Project Page](https://athinagroup.eng.uci.edu/projects/ats-on-the-web/)
- For the Artifact Evaluation, go to [USENIX Security 2023 paper](https://www.usenix.org/conference/usenixsecurity23/presentation/le) and click on the "Abstract PDF" link.

## AutoFR Dataset

The [dataset and its detailed description are available](https://athinagroup.eng.uci.edu/projects/ats-on-the-web/autofr-dataset/). In summary, the
dataset contains 1042 zip files, one per-site. Each zip file
includes the raw collected data of outgoing HTTP requests,
AdGraphs, annotated site snapshots, the action space, filter rules, and more. 

This includes a `Top5k_rules.csv` file that shows all the filter rules created within each zip file. 

**Users must sign a consent form (at the
bottom of the web page) before accessing the dataset.**

## How It Works
AutoFR is the first to balance the trade-off between
blocking ads vs. avoiding visual breakage. The user gives
AutoFR inputs (e.g., the website to generate rules for, and
breakage tolerance threshold w) to AutoFR. It will run our RL
algorithm based on multi-arm bandits and generate filter rules
that block ads while adhering to the given w threshold. 

![AutoFR Implementation](https://bpb-us-e2.wpmucdn.com/faculty.sites.uci.edu/dist/5/718/files/2023/03/AutoFRG_Implementation.png)

AutoFR Example Workflow (Fig. 4 of paper): INITIALIZE (a–c, Alg. 1): (a) spawns n=10 docker instances and visits the site until it finishes loading; (b) extracts the outgoing requests from all visits and builds the action space; (c) extracts the raw graph and annotates it to denote visible ads, images, and text, using JS and Selenium. Once all 10 site snapshots are annotated, we run the RL portion of the AutoFR procedure (steps 1–4). Lastly, AutoFR outputs the filter rules at step 5, e.g., ||s.yimg.com/rq/darla/4-10-0/html/r-sf.html.

For more information, see [Background Information](#background-information).

## Running AutoFR

Follow the instructions below to run AutoFR. [Preview the dependencies](#requirements-and-description).

### Setup

1. We assume you satisfy the [hardware](#hardware-dependencies) and [OS](#os-dependencies) dependencies.
2. Install the core dependencies.
   > $ sudo apt-get install git python3 python3-dev python3-pip
   
   > $ pip3 install virtualenv
    
   1. Install docker using its [official instructions](https://docs.docker.com/engine/install/debian/).

3. > $ git clone https://github.com/UCI-Networking-Group/AutoFR.git
   1. If you are an artifact reviewer, `git checkout artifact-review`
   2. > git submodule update --init --recursive
4. Navigate to the project directory using a terminal window.
5. Create a virtual environment and activate it.
  > $ virtualenv --python=python3 [/save-path/autofrenv] 
> 
  > $ source [/save-path/autofrenv]/bin/activate
6. Install AutoFR dependencies.
  >  $ pip3 install -e .
7. Build the docker container.
> $ docker build -t flg-ad-highlighter-adgraph --build-arg USER_ID=$(id -u) --build-arg GROUP_ID=$(id -g) -f framework-with-ad-highlighter/DockerAdgraphfile .
8. Create output directories that AutoFR expects. See [Understanding the Output](#understanding-the-output) for description.
> $ mkdir temp_graphs; mkdir -p data/output/
9. Done. You are now ready to use AutoFR.   


### Create Filter Rules

1. Make sure you have followed the [setup](#setup) instructions.
2. Open up the AutoFR project directory using a terminal window.
3. Activate your virtual environment.
4. Choose a site that has ads with AdChoice transparency logos. We use https://cricbuzz.com as an example here.
5. Choose how many docker instances you can start in parallel. This depends on the number of cores you have on your system. Pass it using the `--chunk_threshold` argument. Below, we use `6` as an example.
6. > $ python scripts/autofr_controlled.py –site_url
"https://cricbuzz.com" –chunk_threshold 6
7. Filter rules will be presented at the end.

Explore other possible inputs you can give `scripts/auto_controlled.py` by running:
> $ python scripts/auto_controlled.py --help

#### Understanding the Output
Each run of AutoFR will output data into two distinct folders, which are described below. The `data/output` is related to the collection of site snapshots, while `temp_graphs` is related our RL algorithm and outputted filter rules.

Go to `data/output` and go into a folder *AutoFRGControlled\*AdGraph_Snapshots* to see the raw collected data, such as the outgoing HTTP requests, AdGraphs, and site snapshots. 
* **adgraph_networkx**: This holds the site snapshots. (graphml files)
* **init_adgraph_site_feedback/adgraph**: This holds the raw AdGraphs before annotations. (See Sec. 4.1 for how we annotate raw AdGraphs). There will be 10 of these, representing ten visits to the site. (JSON files)
* **init_adgraph_site_feedback/filter_lists**: This holds the rules that we applied (if any). (Text files)
* **init_adgraph_site_feedback/json**: This holds the collected outgoing HTTP requests.
* **init_adgraph_site_feedback/screenshots**: This holds a screenshot of the site. (PNG files)
* **init_adgraph_site_feedback/stats_init.csv**: This holds the counters of ads, images, and text.
* **init_adgraph_site_feedback/log.log**: This holds the verbose logging of the site visit.

Go to `temp_graphs` and go into a folder *AutoFRGControlled_*, which represents a run of the AutoFR. It will contain the outputted filter rules, the action space, and various other information.
* **action_values.csv**: This contains information about the multi-arm bandit run, such as the q-value of each action, the number of pulls per action, and whether we put the arm to sleep or not.
* **dh_graph.json**: This is the hierarchy action space in JSON format. (See Sec. 3.2.1 on how we build the action space.)
* **dh_nodes_history.json**: This just holds more logging information about which actions we took per time step t.
* **final_rules.txt**: These are the filter rules that we generated. (ignore the header comments in this file)
* **low_q_rules.txt**: These are the rules that could block ads but caused breakage beyond the threshold w. (See Sec. 3.2.2 and 3.3)
* **unknown_rules.txt**: These are rules that we put to sleep. (See Sec. 3.2.2 and Algorithm 1 line 23.)
* **log.log**: This holds the verbose logging of our algorithm run.

The output follows our dataset format. See [AutoFR Dataset](#autofr-dataset).

#### Test the Rules In-the-Wild

Test the rules by applying it on the site that you created them for. 

1. Install an adblocker, like Adblock Plus, into your browser (instructions depend on  your browser). 
2. Configure the extension by going to its settings. Turn off all filter lists. 
3. Turn the rules given by AutoFR into per-site rules. For each created rule, append the site it was created for. For instance, if the rule is `||doubleclick.net^` for the site cricbuzz.com, then change it to `||doubleclick.net^$domain=cricbuzz.com`
4. Add in custom rules given by AutoFR (and transform them to be per-site rules). See [further instructions](https://help.adblockplus.org/hc/en-us/articles/360062859913-Add-a-custom-filter).
3. Refresh the site to see if ads are blocked. Note if there is any visual breakage.
4. Remember to undo the changes if you use the adblocker personally.


### Reuse Site Snapshots

As an example of how site snapshots can be reused, we provide the following instructions on reproducing our results from the paper. 

(Note that this only works if you are using the changeset used by the paper and the site snapshots collected during that time. Any changes/improvements to the algorithm of AutoFR or how site snapshots are created will alter the results of the generated filter rules)

1. Make sure you have followed the [setup](#setup) instructions.
2. Open up the AutoFR project directory using a terminal window.
3. Activate your virtual environment.
4. [Get access to our dataset](#autofr-dataset).
5. Download the `Top5K_rules.csv` within our dataset. Open it and choose a zip file to download, keeping track of the site URL as well.
6. Here we assume you chose `AutoFRGEval_www.cricbuzz.com_ad3dce7b.zip`. Unzip the file. 
7. > $ python scripts/autofr_use_snapshots.py --site_url "https://www.cricbuzz.com/"  --snapshot_dir [zip name]/[Snapshots directory]
   * Full example: 
    > $ python scripts/autofr_use_snapshots.py --site_url "https://www.cricbuzz.com/"  --snapshot_dir AutoFRGEval_www.cricbuzz.com_ad3dce7b/AutoFRGControlled_www.cricbuzz.com_AdGraph_Snapshots_82af60e4
8. It should print out the same filter rules as listed in the CSV file for that particular site.
9. (optional) To see the output, go to `temp_graphs` directory and open the newest directory there. See [Understanding the Output](#understanding-the-output) for more information.

For artifact-reviewers and convenience, we also provide a script to automatically check the reproducibility.
1. Make sure you have followed the [setup](#setup) instructions.
2. Open up the AutoFR project directory using a terminal window.
3. Activate your virtual environment.
4. [Get access to our dataset](#autofr-dataset). 
5. Download any zips from the dataset into one directory. (you do not need to unzip it)
6. Download the `Top5K_rules.csv` within our dataset.
7. > $ python scripts/artifact-review/confirm_reproducibility.py --csv_file_path Top5K_rules.csv --snapshots_dir [path to zips]

The [confirm_reproducibility script](scripts/artifact-review/confirm_reproducibility.py) will output a CSV file that summarizes whether the filter rules match. There will also be summary in the console as well.

Example console output:
```text
SUMMARY:
	- Reproduced 3/3
	- Final results in temp_graphs/confirm_reproducible_e400088a.csv
```
## Requirements and Description

### Hardware Dependencies

AutoFR was evaluated using Amazon EC2 instance
m5.2xlarge, which has 8 cores, 32 GiB of memory, 35 GiB
of storage, and up to 10 Gbps of network bandwidth. We
recommend something similar, going as low as 16 GiB of
memory with 20 GiB of storage. 

**Limitations**: Currently, AutoFR does not work for M1 Macbooks. This is a work in progress. 

### Software Dependencies

We list the dependencies that are necessary to run AutoFR. Please follow instructions in [Setup section](#setup) instead to install them.

#### OS Dependencies

AutoFR has been tested on Debian 5.10 and Ubuntu 18.04.6 LTS.

#### Core Dependencies
* Python 3.6+
* git
* pip3 
* virtualenv (or conda)
* docker

#### Python Dependencies

* tldextract
* networkx
* adblockparser
* pandas
* numpy
* selenium
* pyvirtualdisplay

#### Prior Work Dependencies

AutoFR integrates browser extensions and an instrumented browser:
* [Ad Highlighter](https://github.com/UCI-Networking-Group/ad-highlighter-autofr): a browser extension that detects iframe ads based on AdChoice logos.
* [AdGraph](https://github.com/uiowa-irl/AdGraph): an instrumented Chromium browser that generates a raw graph representation of how the site is loaded.
* [Adblock Plus](https://gitlab.com/adblockinc/ext/adblockplus/adblockplusui): a browser extension that blocks HTTP requests using filter rules, and more...

## Background Information

To further understand our system, we describe some important terminology below and reference the paper when needed.

* **Filter Rules**: AutoFR focuses on filter rules that block HTTP requests to remove ads while minimizing visual breakage (such as missing images and text) for websites. Example of rules with different granularity: `||example.com^` , `||ads.example.com^`, `||example.com/ads.js`. 
* **Site Snapshots**: Graph representations of how a site is loaded. The nodes represent HTML elements, JS scripts, HTTP requests. The edges represent whether HTML elements are connected based on HTML structure, if JS scripts initiated a request, if HTML elements initiated a request, if JS scripts created a HTML element, etc... See Sec 4.1 and Fig. 5.
* **Threshold w**: It is a design parameter that helps AutoFR balances the trade-off between blocking ads vs. avoiding visual breakage. It ranges from 0 to 1. The higher values represent the user wanting to avoid breakage at the cost of not creating any filter rules. See Sec. 3 and 3.3.2.
* **Breakage**: In our case, breakage means that after a filter rule blocks some HTTP requests, some legitimate images and text of a website may be missing. For instance, for a news site, breakage would entail missing article titles, descriptions, and images. See Eq. (2) of paper.
* **Detecting Visual Components**: AutoFR relies on the detection of ads, images, and text on a website. For ads, we rely on Ad Highlighter (see [Prior Work Dependencies](#prior-work-dependencies)). For images and text, we write our own custom JS to walk the HTML DOM and identify elements with <img> tags or CSS `background-url`. To determine visibility, we look at whether the element's width and height are > 2px and its opacity > 0.1. See Sec. 4.3.

## Disclaimer

The web changes naturally. AutoFR is only as good as its components. Thus, if a site does not serve ads that Ad Highlighter can detect or use obfuscation techniques, then AutoFR may not be able to generate rules for the given site. See Sec. 5.3.4 and 4.3. There may be other factors, such as *w* being too high to generate rules for, etc... Over time, AutoFR will improve as we maintain it, but we cannot guarantee that it will work on every website.


## Citation
If you create a publication using AutoFR, please cite the corresponding paper as follows:
```
@inproceedings{le2023autofr,
  title     = {{AutoFR: Automated Filter Rule Generation for Adblocking}},
  author    = {Le, Hieu and Elmalaki, Salma and Markopoulou, Athina and Shafiq, Zubair},
  booktitle = {Proceedings of the 32nd USENIX Security Symposium (USENIX Security)},
  year      = {2023},
  month     = aug,
  address   = {Anaheim, CA}
}
```
We also encourage you to provide us ([athinagroupreleases@gmail.com](mailto:athinagroupreleases@gmail.com)) with a link to your publication. We use this information in reports to our funding agencies.

## Contact Us

Feel free to contact the authors, specifically [Hieu Le](https://levanhieu.com) if you have any questions.

## Acknowledgements

To integrate [AdGraph](https://github.com/uiowa-irl/AdGraph) successfully, we thank its authors ([Umar Iqbal](https://umariqbal.com/)), who  graciously provided the necessary code to help parse AdGraphs. We include it in [adgraphapi](adgraphapi) with slight modifications.
