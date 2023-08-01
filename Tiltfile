load('ext://restart_process', 'docker_build_with_restart')
load('ext://cert_manager', 'deploy_cert_manager')


def deploy_cert_manager_if_needed():
    cert_manager_var = '__CERT_MANAGER__'
    if os.getenv(cert_manager_var) != '1':
        deploy_cert_manager(version="v1.12.3")
        os.putenv(cert_manager_var, '1')


# Set up our build helper image that has delve in it. We use a helper so parallel image builds don't all simultaneously
# install delve. Instead, they all wait for this build to complete, and then proceed in parallel.
docker_build(
    ref='helper',
    context='.',
    build_args={'GO_VERSION': '1.20'},
    dockerfile_contents='''
ARG GO_VERSION
FROM golang:$GO_VERSION
RUN CGO_ENABLED=0 go install github.com/go-delve/delve/cmd/dlv@v$GO_VERSION
'''
)


def build_binary(repo, binary, image, debug=True):
    gcflags = ''
    if debug:
        gcflags = "-gcflags 'all=-N -l'"

    # Treat the main binary as a local resource, so we can automatically rebuild it when any of the deps change. This
    # builds it locally, targeting linux, so it can run in a linux container.
    local_resource(
        '{}_{}_binary'.format(repo, binary),
        cmd='''
cd ../{repo}
mkdir -p .tiltbuild/bin
CGO_ENABLED=0 GOOS=linux go build {gcflags} -o .tiltbuild/bin/{binary} ./cmd/{binary}
'''.format(repo=repo, binary=binary, gcflags=gcflags),
        deps=['api', 'cmd/{}'.format(binary), 'internal', 'pkg', 'go.mod', 'go.sum']
    )

    entrypoint = '/{}'.format(binary)
    if debug:
        entrypoint = '/dlv --accept-multiclient --api-version=2 --headless=true --listen :30000 exec --continue -- ' + entrypoint

    # Configure our image build. If the file in live_update.sync (.tiltbuild/bin/$binary) changes, Tilt
    # copies it to the running container and restarts it.
    docker_build_with_restart(
        # This has to match an image in the k8s_yaml we call below, so Tilt knows to use this image for our Deployment,
        # instead of the actual image specified in the yaml.
        ref='{image}:{binary}'.format(image=image, binary=binary),
        # This is the `docker build` context, and because we're only copying in the binary we've already had Tilt build
        # locally, we set the context to the directory containing the binary.
        context='../{}/.tiltbuild/bin'.format(repo),
        # We use a slimmed-down Dockerfile that only has $binary in it.
        dockerfile_contents='''
FROM gcr.io/distroless/static:debug
WORKDIR /
COPY --from=helper /go/bin/dlv /
COPY {} /
        '''.format(binary),
        # The set of files Tilt should include in the build. In this case, it's just the binary we built above.
        only=binary,
        # If .tiltbuild/bin/$binary changes, Tilt will copy it into the running container and restart the process.
        live_update=[
            sync('.tiltbuild/bin/{}'.format(binary), '/{}'.format(binary)),
        ],
        # The command to run in the container.
        entrypoint=entrypoint,
    )


def process_yaml(yaml):
    if type(yaml) == 'string':
        objects = read_yaml_stream(yaml)
    elif type(yaml) == 'blob':
        objects = decode_yaml_stream(yaml)
    else:
        fail('expected a string or blob, got: {}'.format(type(yaml)))

    for o in objects:
        # For Tilt's live_update functionality to work, we have to run the container as root. Remove any PSA labels
        # to allow this.
        if o['kind'] == 'Namespace' and 'labels' in o['metadata']:
            labels_to_delete = [label for label in o['metadata']['labels'] if label.startswith('pod-security.kubernetes.io')]
            for label in labels_to_delete:
                o['metadata']['labels'].pop(label)

        if o['kind'] != 'Deployment':
            # We only need to modify Deployments, so we can skip this
            continue

        # For Tilt's live_update functionality to work, we have to run the container as root. Otherwise, Tilt won't
        # be able to untar the updated binary in the container's file system (this is how live update
        # works). If there are any securityContexts, remove them.
        if "securityContext" in o['spec']['template']['spec']:
            o['spec']['template']['spec'].pop('securityContext')
        for c in o['spec']['template']['spec']['containers']:
            if "securityContext" in c:
                c.pop('securityContext')

        # If multiple Deployment manifests all use the same image but use different entrypoints to change the binary,
        # we have to adjust each Deployment to use a different image. Tilt needs each Deployment's image to be
        # unique. We replace the tag with what is effectively :$binary, e.g. :helm.
        for c in o['spec']['template']['spec']['containers']:
            if c['name'] == 'kube-rbac-proxy':
                continue

            command = c['command'][0]
            if command.startswith('./'):
                command = command.removeprefix('./')
            elif command.startswith('/'):
                command = command.removeprefix('/')

            image_without_tag = c['image'].rsplit(':', 1)[0]

            # Update the image so instead of :$tag it's :$binary
            c['image'] = '{}:{}'.format(image_without_tag, command)

    # Now apply all the yaml
    k8s_yaml(encode_yaml_stream(objects))


# data format:
# {
#     'image': 'quay.io/operator-framework/rukpak',
#     'yaml': 'manifests/overlays/cert-manager',
#     'binaries': {
#         'core': 'core',
#         'crdvalidator': 'crd-validation-webhook',
#         'helm': 'helm-provisioner',
#         'webhooks': 'rukpak-webhooks',
#     },
# },
def deploy_repo(repo, data):
    print('Deploying repo {}'.format(repo))
    deploy_cert_manager_if_needed()

    local_port = data['starting_debug_port']
    for binary, deployment in data['binaries'].items():
        build_binary(repo, binary, data['image'])
        k8s_resource(deployment, port_forwards=['{}:30000'.format(local_port)])
        local_port += 1
    process_yaml(kustomize('../{}/{}'.format(repo, data['yaml'])))
