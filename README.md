# DataTrove

DataTrove is a library to process, filter and deduplicate text data at a very large scale. It provides a set of prebuilt commonly used processing blocks with a framework to easily add custom functionality.

DataTrove processing pipelines are platform-agnostic, running out of the box locally or on a slurm cluster. Its (relatively) low memory usage and multiple step design makes it ideal for large workloads, such as to process an LLM's training data.

Local, remote and other file systems are supported through [fsspec](https://filesystem-spec.readthedocs.io/en/latest/).

## Installation

```bash
git clone git@github.com:huggingface/datatrove.git && cd datatrove
pip install -e ".[FLAVOUR]"
```
Available flavours (combine them with `,` i.e. `[processing,s3]`:
- `all` installs everything
- `io` dependencies to read `warc/arc/wet` files and arrow/parquet formats
- `processing` dependencies for text extraction, filtering and tokenization
- `s3` s3 support
- `cli` for command line tools

## Quickstart examples
You can check the following [examples](examples):
- [process_common_crawl_dump.py](examples/process_common_crawl_dump.py) full pipeline to read commoncrawl warc files, extract their text content, filters and save the resulting data to s3. Runs on slurm
- [tokenize_c4.py](examples/tokenize_c4.py) reads data directly from huggingface's hub to tokenize the english portion of the C4 dataset using the `gpt2` tokenizer
- [minhash_deduplication.py](examples/minhash_deduplication.py) full pipeline to run minhash deduplication of text data
- [sentence_deduplication.py](examples/sentence_deduplication.py) example to run sentence level exact deduplication
- [exact_substrings.py](examples/exact_substrings.py) example to run ExactSubstr (requires [this repo](https://github.com/google-research/deduplicate-text-datasets)

## Pipeline
### DataTrove Document
Each pipeline block processes data in the datatrove [`Document`](src/datatrove/data.py) format:
- `text` the actual text content for each sample
- `id` a unique id (string) for this sample
- `metadata` a dictionary where any additional info may be stored

### Types of pipeline blocks
Each pipeline block takes a generator of `Document` as input and returns another generator of `Document`.
- **[readers](src/datatrove/pipeline/readers)** read data from different formats and yield `Document`
- **[writers](src/datatrove/pipeline/writers)** save `Document` to disk/cloud in different formats
- **[extractors](src/datatrove/pipeline/extractors)** extract text content from raw formats (such as webpage html)
- **[filters](src/datatrove/pipeline/filters)** filter out (remove) some `Document`s based on specific rules/criteria
- **[stats](src/datatrove/pipeline/stats)** blocks to collect statistics on the dataset
- **[tokens](src/datatrove/pipeline/tokens)** blocks to tokenize data or count tokens
- **[dedup](src/datatrove/pipeline/dedup)** blocks for deduplication
### Full pipeline
A pipeline is defined as a list of pipeline blocks. As an example, the following pipeline would read data from disk, randomly filter (remove) some documents and write them back to disk:
```python
from datatrove.pipeline.readers import CSVReader
from datatrove.pipeline.filters import SamplerFilter
from datatrove.pipeline.writers import JsonlWriter

pipeline = [
    CSVReader(
        data_folder="/my/input/path"
    ),
    SamplerFilter(rate=0.5),
    JsonlWriter(
        output_folder="/my/output/path"
    )
]
```

## Executors
Pipelines are platform-agnostic, which means that the same pipeline can smoothly run on different execution environments without any changes to its steps. Each environment has its own PipelineExecutor.
Some options common to all executors:
- `pipeline` a list consisting of the pipeline steps that should be run
- `logging_dir` a datafolder where log files, statistics and more should be saved
- `skip_completed` (_bool_, `True` by default) datatrove keeps track of completed tasks so that when you relaunch a job they can be skipped. Set this to `False` to disable this behaviour

Call an executor's `run` method to execute its pipeline.


### LocalPipelineExecutor
This executor will launch a pipeline on a local machine.
Options:
- `tasks` total number of tasks to run
- `workers` how many tasks to run simultaneously. If `-1`, no limit. Anything `> 1` will use multiprocessing to execute the tasks.
- `start_method` method to use to spawn a multiprocessing Pool. Ignored if `workers` is 1

<details>
  <summary>Example executor</summary>

```python
from datatrove.executor import LocalPipelineExecutor
executor = LocalPipelineExecutor(
    pipeline=[
        ...
    ],
    logging_dir="logs/",
    tasks=10,
    workers=5
)
executor.run()
```
</details>

### SlurmPipelineExecutor
This executor will launch a pipeline on a slurm cluster, using slurm job arrays to group and manage tasks.
Options:
- `tasks` total number of tasks to run. **required**
- `time` slurm time limit string. **required**
- `partition` slurm partition. **required**
- `workers` how many tasks to run simultaneously. If `-1`, no limit. Slurm will run `workers` tasks at a time. (default: `-1`)
- `job_name` slurm job name (default: "data_processing)
- `depends` another SlurmPipelineExecutor instance, which will be a dependency of this pipeline (current pipeline will only start executing after the depended on pipeline successfully completes)
- `sbatch_args` dictionary with any other arguments you would like to pass to sbatch
- `slurm_logs_folder` where to save the slurm log files. If using a local path for `logging_dir`, they will be saved on `logging_dir/slurm_logs`. If not, they will be saved as a subdir of the current directory.
<details>
  <summary>Other options</summary>

- `cpus_per_task` how many cpus to give each task (default: `1`)
- `qos` slurm qos (default: "normal")
- `mem_per_cpu_gb` memory per cpu, in GB (default: 2)
- `env_command` custom command to activate a python environment, if needed
- `condaenv` conda environment to activate
- `venv_path` path to a python environment to activate
- `max_array_size` the _MaxArraySize_ value in `$ scontrol show config`. If number of tasks exceeds this number, it will split into multiple array jobs (default: 1001)
- `max_array_launch_parallel` if we need multiple jobs due to max_array_size, whether to launch them all in one go (parallel) or sequentially (default: `False`)
- `stagger_max_array_jobs` when max_array_launch_parallel is True, this determines how many seconds to wait between launching each of the parallel jobs (default: `0`)
- `run_on_dependency_fail` start executing when a job we depend on finishes even if it has failed (default: `False`)
- `randomize_start` randomize the start of each task in a job in a ~3 min window. Useful when heavily hitting an s3 bucket for example. (default: `False`)
</details>
<details>
  <summary>Example executor</summary>

```python
from datatrove.executor import SlurmPipelineExecutor
executor1 = SlurmPipelineExecutor(
    pipeline=[
        ...
    ],
    job_name="my_cool_job1",
    logging_dir="logs/job1",
    tasks=500,
    workers=100,  # omit to run all at once
    time="10:00:00",  # 10 hours
    partition="hopper-cpu"
)
executor2 = SlurmPipelineExecutor(
    pipeline=[
        ...
    ],
    job_name="my_cool_job2",
    logging_dir="logs/job2",
    tasks=1,
    time="5:00:00",  # 5 hours
    partition="hopper-cpu",
    depends=executor1  # this pipeline will only be launched after executor1 successfuly completes
)
# executor1.run()
executor2.run() # this will actually launch executor1, as it is a dependency, so no need to launch it explicitly
```
</details>

## Logging
For a pipeline with `logging_dir` **mylogspath/exp1**, the following folder structure would be created:

<details>
  <summary>See folder structure</summary>

```
└── mylogspath/exp1
    │── executor.json ⟵ json dump of the executor options and pipeline steps
    │── launch_script.slurm ⟵ the slurm config created and used to launch this job (if running on slurm)
    │── executor.pik ⟵ the slurm config created and used to launch this job (if running on slurm)
    │── ranks_to_run.json ⟵ list of tasks that are being run
    │── logs/
    │   └──[task_00000.log, task_00001.log, task_00002.log, ...] ⟵ individual logging files for each task
    │── completions/
    │   └──[00004, 00007, 00204, ...] ⟵ empty files marking a task as completed. Using when relaunching/resuming a job (only unfinished tasks will be run)
    │── stats/
    │   └──[00000.json, 00001.json, 00002.json, ...] ⟵ individual stats for each task (number of samples processed, filtered, removed, etc)
    └── stats.json ⟵ global stats from all tasks
```
</details>

## DataFolder / paths
Datatrove supports a wide variety of input/output sources through [fsspec](https://filesystem-spec.readthedocs.io/en/latest/).

There are a few ways to provide a path to a datatrove block (for `input_folder`, `logging_dir`, `data_folder` and so on arguments):
- `str`: the simplest way is to pass a single string. Example: `/home/user/mydir`, `s3://mybucket/myinputdata`, `hf://datasets/allenai/c4/en/`

- `(str, fsspec filesystem instance)`: a string path and a fully initialized filesystem object. Example: `("s3://mybucket/myinputdata", S3FileSystem(client_kwargs={"endpoint_url": endpoint_uri}))`
- `(str, dict)`: a string path and a dictionary with options to initialize a fs. Example (equivalent to the previous line): `("s3://mybucket/myinputdata", {"client_kwargs": {"endpoint_url": endpoint_uri}})`
- `DataFolder`: you can initialize a [DataFolder](src/datatrove/io.py) object directly and pass it as an argument

Under the hood these argument combinations are parsed by [`get_datafolder`](src/datatrove/io.py#116).

## Practical guides

### Reading data
Usually, pipelines will start with a [Reader](src/datatrove/pipeline/readers) block.
Most readers take a `data_folder` argument — a path to a folder containing the data to be read.

These files will be distributed across each task. If you have `N` tasks, task with rank `i` (0-based) will process files `i, i+N, i+2N, i+3N,...`.

Internally, each reader reads data and converts it into a dictionary before creating a `Document` object.

Some options common to most readers:
- `text_key` the dictionary key containing the text content for each sample. Default: `text`
- `id_key` the dictionary key containing the id for each sample. Default: `id`
- `default_metadata` a dictionary for any default metadata values you would like to add (such as their source, for example)
- `recursive` whether to look for files recursively in `data_folder`'s subdirectories
- `glob_pattern` use this field to match specific files. For instance, `glob_pattern="*/warc/*.warc.gz"` will match files with a `.warc.gz` file extension on the `warc/` folder of each of the `data_folder`'s subdirectories
- `adapter` this function takes the raw dictionary obtained from the reader and returns a dictionary with `Document`'s field names. You may overwrite this function ([_default_adapter](src/datatrove/pipeline/readers/base.py)) if you would like.
- `limit` read only a certain number of samples. Useful for testing/debugging

### Extracting text
You can use [extractors](src/datatrove/pipeline/extractors) to extract text content from raw html. The most commonly used extractor in datatrove is [Trafilatura](src/datatrove/pipeline/extractors/trafilatura.py), which uses the [trafilatura](https://trafilatura.readthedocs.io/en/latest/) library.

### Filtering data
[Filters](src/datatrove/pipeline/filters) are some of the most important blocks of any data processing pipeline. Datatrove's filter blocks take a `Document` and return a boolean (`True` to keep a document, `False` to remove it). Removed samples do not continue to the next pipeline stage. You can also save the removed samples to disk by passing a [Writer](src/datatrove/pipeline/writers) to the `excluded_writer` parameter.

### Saving data
Once you are done processing your data you will probably want to save it somewhere. For this you can use a [writer](src%2Fdatatrove%2Fpipeline%2Fwriters%2Fjsonl.py).
Writers require an `output_folder` (the path where data should be saved). You can choose the `compression` to use (default: `gzip`) and the filename to save each file as.
For the `output_filename`, a template is applied using the following arguments:
- `${rank}` replaced with the current task's rank. Note that if this tag isn't present, **different tasks may try to write to the same location**
- `${id}` replaced with the sample id
- metadata: any other `${tag}` will be replaced with the corresponding `document.metadata['tag']` value

An example to separate samples by language based on their `lang` metadata field:
```
JsonlWriter(
    f"{MAIN_OUTPUT_PATH}/non_english/",
    output_filename="${language}/" + DUMP + "/${rank}.jsonl.gz",  # folder structure: language/dump/file
)
```

### Deduplicating data
For deduplication check the examples [minhash_deduplication.py](examples/minhash_deduplication.py), [sentence_deduplication.py](examples/sentence_deduplication.py) and [exact_substrings.py](examples/exact_substrings.py).

## Contributing

```bash
pip install -e ".[dev]"
```

Install pre-commit code style hooks:
```bash
pre-commit install
```

Run the tests:
```bash
pytest -sv ./tests/ 
```
