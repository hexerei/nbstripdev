# nbstripdev

Just a test repository to test JQ's functionality to strip Jupyter Notebook files before being committed to repository.
In detail I am using the jq command-line JSON processor to strip output from notebook files and reduce metadata to known output, 
so notebook files can be committed to a repository without unnecessary output, which reduces size, increases security and ensures
that different developers can work on the same notebook files without interferring of differences in python version or different 
output in the notebooks.

jq command-line processor is available at: https://stedolan.github.io/jq/

## Solution via Command-Line

Jupyter Notebook allows executing a terminal window from which the user could commit the notebooks directly via git command-line. 
The advantage is the ease of implementation and flexibility for the developer in defining branch and repository to commit to, 
but still we would need to define an automated process to strip metadata and output.

Here we make use of **jq** (https://stedolan.github.io/jq/) a lightweight and flexible command-line JSON processor. 
Jq has it's own filter and query language, but is well documented and easy to use. A possible invocation to filter output and metadata could be:

    jq --indent 1 \
        '(.cells[] | select(has("outputs")) | .outputs) = []
        | (.cells[] | select(has("execution_count")) | .execution_count) = null
        | .metadata = {"language_info": {"name":"python", "pygments_lexer": "ipython3"}}
        | .cells[].metadata = {}' my-notebook.ipynb

Each line within the single-quotes defines a filter applied to the Notebook, in this case stripping the output and metadata from the cells 
and sets the overall metadata to a diff-friendly format. Of course this is just one possible solution and could be adapted after deeper research.

The advantage of jq is, that it not only fully strips all unwanted content from our notebook, but it is also blazing fast, as it is implemented in C.

Of course this is a lot of typing for manual invocation which could be omitted by creating an alias in _.bashrc_.  

    alias nbstrip="jq --indent 1 \
        '(.cells[] | select(has(\"outputs\")) | .outputs) = []  \
        | (.cells[] | select(has(\"execution_count\")) | .execution_count) = null  \
        | .metadata = {\"language_info\": {\"name\": \"python\", \"pygments_lexer\": \"ipython3\"}} \
        | .cells[].metadata = {} \
        '"

Then it can be simply invoked by:

    nbstrip  my-notebook.ipynb > stripped.ipynb

### Automation: Integrating with Git

To make the stripping of Notebooks default for all repositories, we can use gitattributes filter option to configure automatic stripping on commit and diff.

Following would go into _~/.gitconfig_ file:

    [core]
    attributesfile = ~/.gitattributes_global

    [filter "nbstrip_full"]
    clean = "jq --indent 1 \
            '(.cells[] | select(has(\"outputs\")) | .outputs) = []  \
            | (.cells[] | select(has(\"execution_count\")) | .execution_count) = null  \
            | .metadata = {\"language_info\": {\"name\": \"python\", \"pygments_lexer\": \"ipython3\"}} \
            | .cells[].metadata = {} \
            '"
    smudge = cat
    required = true

and then in _~/.gitattributes_global_ file:

    *.ipynb filter=nbstrip_full

Of course this can be also set in repository specific _.gitattributes_ file to achieve a more granual implementation.

Once this is implemented existing Notebooks would have to added to a 'do nothing' commit to allow the strip filter to execute. 
Future Notebooks will be automatically stripped.

**Note** that unchanged Notebooks that are re-run, could cause a false positive on git-diff, even though the stripped 
version does not differ, due to the timestamp being updated on re-run. Also external diff tools do not apply git filters 
and therefore would show unwanted differences.

To overcome this you could add a function in _.bashrc_ to strip all Notebooks in your directory:

    function nbstrip_all {
        for nbfile in *.ipynb; do
            echo "$( nbstrip $nbfile )" > $nbfile
        done
        unset nbfile
    }

**Note** clean and smudge filters from git often don't work well with rebase operations, therefore before rebasing the user 
would need to comment out the filter definitions in the _.gitattributes_global_ file while performing a rebase 
and then uncommenting them again when done.
