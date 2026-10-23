load("@bazel_skylib//rules:diff_test.bzl", "diff_test")
load("@bazel_lib//lib:params_file.bzl", "params_file")
load("@bazel_lib//lib:transitions.bzl", "platform_transition_filegroup")
load("@container_structure_test//:defs.bzl", "container_structure_test")
load("@rules_distroless//distroless:defs.bzl", "cacerts", "group", "passwd")
load("@rules_oci//oci:defs.bzl", "oci_image", "oci_load")
load("@tar.bzl", "tar")

COMPATIBLE_WITH = select({
    "@platforms//cpu:x86_64": ["@platforms//cpu:x86_64"],
    "@platforms//cpu:arm64": ["@platforms//cpu:arm64"],
}) + [
    "@platforms//os:linux",
]

passwd(
    name = "passwd",
    entries = [
        {
            "uid": 0,
            "gid": 0,
            "home": "/root",
            "shell": "/bin/bash",
            "username": "r00t",
        },
        {
            "uid": 100,
            "gid": 65534,
            "home": "/home/_apt",
            "shell": "/usr/sbin/nologin",
            "username": "_apt",
        },
    ],
)

group(
    name = "group",
    entries = [
        {
            "name": "root",
            "gid": 0,
        },
        {
            "name": "_apt",
            "gid": 65534,
        },
    ],
)

tar(
    name = "sh",
    mtree = [
        # needed as dpkg assumes sh is installed in a typical debian installation.
        "./bin/sh type=link link=/bin/bash",
    ],
)

cacerts(
    name = "cacerts",
    package = "@bullseye//ca-certificates:data",
    target_compatible_with = COMPATIBLE_WITH,
)

oci_image(
    name = "image",
    architecture = select({
        "@platforms//cpu:arm64": "arm64",
        "@platforms//cpu:x86_64": "amd64",
    }),
    env = {
        # Required to use the SSL certs from `cacerts()`
        "SSL_CERT_FILE": "/etc/ssl/certs/ca-certificates.crt",
    },
    os = "linux",
    # NOTE: this is needed because, otherwise, bazel test //... fails, even
    # when container_structure_test already has target_compatible_with.
    # See 136
    target_compatible_with = COMPATIBLE_WITH,
    tars = [
        # This target contains all the installed packages.
        "@bullseye//:flat",
        ":sh",
        ":passwd",
        ":group",
        ":cacerts",
    ],
)

platform(
    name = "linux_arm64",
    constraint_values = [
        "@platforms//os:linux",
        "@platforms//cpu:arm64",
    ],
)

platform(
    name = "linux_amd64",
    constraint_values = [
        "@platforms//os:linux",
        "@platforms//cpu:x86_64",
    ],
)

platform_transition_filegroup(
    name = "image_platform",
    srcs = [":image"],
    target_platform = select({
        "@platforms//cpu:arm64": ":linux_arm64",
        "@platforms//cpu:x86_64": ":linux_amd64",
    }),
)

oci_load(
    name = "tarball",
    image = ":image_platform",
    repo_tags = [
        "distroless/test:latest",
    ],
    # NOTE: this is needed because, otherwise, bazel test //... fails, even
    # when container_structure_test already has target_compatible_with.
    # See 136
    target_compatible_with = COMPATIBLE_WITH,
)

container_structure_test(
    name = "test",
    configs = select({
        "@platforms//cpu:arm64": ["test_linux_arm64.yaml"],
        "@platforms//cpu:x86_64": ["test_linux_amd64.yaml"],
    }),
    image = ":image_platform",
    target_compatible_with = COMPATIBLE_WITH,
)

# With `include_transitive = False`, a package's target must contain nothing but
# its own data, i.e. none of jq's dependencies (libjq1, libonig5, ...).
# Only the rootpaths are compared, so no .deb needs to be built.
platform_transition_filegroup(
    name = "jq_nodeps",
    srcs = ["@bullseye_nodeps//jq"],
    target_platform = select({
        "@platforms//cpu:arm64": ":linux_arm64",
        "@platforms//cpu:x86_64": ":linux_amd64",
    }),
)

platform_transition_filegroup(
    name = "jq_nodeps_data",
    srcs = ["@bullseye_nodeps//jq:data"],
    target_platform = select({
        "@platforms//cpu:arm64": ":linux_arm64",
        "@platforms//cpu:x86_64": ":linux_amd64",
    }),
)

[
    params_file(
        name = name + "_rootpaths",
        out = name + "_rootpaths.txt",
        args = ["$(rootpaths :%s)" % name],
        data = [":" + name],
    )
    for name in [
        "jq_nodeps",
        "jq_nodeps_data",
    ]
]

diff_test(
    name = "include_transitive_false_test",
    file1 = ":jq_nodeps_rootpaths",
    file2 = ":jq_nodeps_data_rootpaths",
)
