TaskCluster Task-Graph Generation:

    Task Kind - Tasks are grouped by kind, where tasks of the same kind have substantial similarities or share common processing logic. Kinds are the primary means of supporting diversity, in that a developer can add a new kind to do just about anything without impacting other kinds.

    Task Attributes - Tasks have string attributes by which can be used for filtering. 

    Task Labels - Each task has a unique identifier within the graph that is stable across runs of the graph generation algorithm. Labels are replaced with TaskCluster TaskIds at the latest time possible, facilitating analysis of graphs without distracting noise from randomly-generated taskIds.

    Optimization - replacement of a task in a graph with an equivalent, already-completed task, or a null task, avoiding repetition of work.

Kinds:

    They provide an interface between the large-scale graph-generation process and the small-scale task-definition needs of different kinds of tasks. 

    A kind.yml file contains data about the kind, as well as referring to a Python class implementing the kind in its implementation key.

    `kind.yml` example:


```
    loader: taskgraph.loader.single_dep:loader

    transforms:
        - taskgraph.transforms.name_sanity:transforms
        - taskgraph.transforms.balrog_submit:transforms
        - taskgraph.transforms.scriptworker:add_balrog_scopes
        - taskgraph.transforms.task:transforms

    kind-dependencies:
        - beetmover-repackage

    only-for-attributes:
        - nightly
        - shippable

    not-for-build-platforms:
        - android-api-16-nightly/opt
        - android-x86_64-nightly/opt
        - android-x86-nightly/opt
        - android-aarch64-nightly/opt

    job-template:
        update-no-wnp:
            by-release-type:
                nightly.*: false
                default: true
```

    `balrog_submit:transforms` py example:

```
from __future__ import absolute_import, print_function, unicode_literals

from taskgraph.loader.single_dep import schema
from taskgraph.transforms.base import TransformSequence
from taskgraph.util.attributes import copy_attributes_from_dependent_job
from taskgraph.util.schema import (
    optionally_keyed_by, resolve_keyed_by,
)
from taskgraph.util.scriptworker import (
    get_balrog_server_scope, get_worker_type_for_scope
)
from taskgraph.util.treeherder import replace_group
from taskgraph.transforms.task import task_description_schema
from voluptuous import Optional


balrog_description_schema = schema.extend({
    # unique label to describe this balrog task, defaults to balrog-{dep.label}
    Optional('label'): basestring,


    Optional(
        'update-no-wnp',
        description="Whether the parallel `-No-WNP` blob should be updated as well.",
    ): optionally_keyed_by('release-type', bool),

    # treeherder is allowed here to override any defaults we use for beetmover.  See
    # taskcluster/taskgraph/transforms/task.py for the schema details, and the
    # below transforms for defaults of various values.
    Optional('treeherder'): task_description_schema['treeherder'],

    Optional('attributes'): task_description_schema['attributes'],

    # Shipping product / phase
    Optional('shipping-product'): task_description_schema['shipping-product'],
    Optional('shipping-phase'): task_description_schema['shipping-phase'],
})


transforms = TransformSequence()
transforms.add_validate(balrog_description_schema)


@transforms.add
def handle_keyed_by(config, jobs):
    """Resolve fields that can be keyed by platform, etc."""
    fields = [
        "update-no-wnp",
    ]
    for job in jobs:
        label = job.get('dependent-task', object).__dict__.get('label', '?no-label?')
        for field in fields:
            resolve_keyed_by(
                item=job, field=field, item_name=label,
                **{
                    'project': config.params['project'],
                    'release-type': config.params['release_type'],
                }
            )
        yield job


@transforms.add
def make_task_description(config, jobs):
    for job in jobs:
        dep_job = job['primary-dependency']

        treeherder = job.get('treeherder', {})
        treeherder.setdefault('symbol', 'c-Up(N)')
        dep_th_platform = dep_job.task.get('extra', {}).get(
            'treeherder', {}).get('machine', {}).get('platform', '')
        treeherder.setdefault('platform',
                              "{}/opt".format(dep_th_platform))
        treeherder.setdefault(
            'tier',
            dep_job.task.get('extra', {}).get('treeherder', {}).get('tier', 1)
        )
        treeherder.setdefault('kind', 'build')

        attributes = copy_attributes_from_dependent_job(dep_job)

        treeherder_job_symbol = dep_job.task['extra']['treeherder']['symbol']
        treeherder['symbol'] = replace_group(treeherder_job_symbol, 'c-Up')

        if dep_job.attributes.get('locale'):
            attributes['locale'] = dep_job.attributes.get('locale')

        label = job['label']

        description = (
            "Balrog submission for locale '{locale}' for build '"
            "{build_platform}/{build_type}'".format(
                locale=attributes.get('locale', 'en-US'),
                build_platform=attributes.get('build_platform'),
                build_type=attributes.get('build_type')
            )
        )

        upstream_artifacts = [{
            "taskId": {"task-reference": "<beetmover>"},
            "taskType": "beetmover",
            "paths": [
                "public/manifest.json"
            ],
        }]

        server_scope = get_balrog_server_scope(config)

        task = {
            'label': label,
            'description': description,
            'worker-type': get_worker_type_for_scope(config, server_scope),
            'worker': {
                'implementation': 'balrog',
                'upstream-artifacts': upstream_artifacts,
                'balrog-action': 'submit-locale',
                'suffixes': ['', '-No-WNP'] if job.get('update-no-wnp') else [''],
            },
            'dependencies': {'beetmover': dep_job.label},
            'attributes': attributes,
            'run-on-projects': dep_job.attributes.get('run_on_projects'),
            'treeherder': treeherder,
            'shipping-phase': job.get('shipping-phase', 'promote'),
            'shipping-product': job.get('shipping-product'),
        }

        yield task

```
