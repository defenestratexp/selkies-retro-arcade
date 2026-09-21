// Builds the retro-arcade image and pushes it to ECR. Deployment is a separate
// concern, handled by the Ansible role in ../ansible.
//
// Expects: an agent labelled 'ops' with docker, and a Jenkins AWS credential
// with id 'ecr-credentials' that can push to ${ECR_REPO}.
pipeline {
    agent { label 'ops' }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10', daysToKeepStr: '30'))
        // Base image is a full Ubuntu desktop plus Wine with i386 multiarch --
        // the first build pulls and installs a lot.
        timeout(time: 45, unit: 'MINUTES')
    }

    environment {
        ECR_REGISTRY = '123456789012.dkr.ecr.us-west-2.amazonaws.com'
        ECR_REPO     = 'homelab/retro-arcade'
        AWS_REGION   = 'us-west-2'
        IMAGE_TAG    = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Build image') {
            steps {
                dir('image') {
                    sh '''
                        docker build -t ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG} .
                        docker tag ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG} ${ECR_REGISTRY}/${ECR_REPO}:latest
                    '''
                }
            }
        }

        stage('Smoke test') {
            steps {
                // Cheap guard against the things most likely to be wrong:
                // that 32-bit Wine actually landed, and that the tooling exists.
                sh '''
                    docker run --rm --entrypoint sh ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG} -c '
                        set -e
                        for b in wine scummvm dosbox-x; do
                            p=$(command -v $b) || { echo "MISSING: $b (PATH=$PATH)"; exit 1; }
                            echo "  found $b -> $p"
                        done
                        dpkg --print-foreign-architectures | grep -q i386 || { echo "i386 multiarch missing -- 32-bit Windows apps will not run"; exit 1; }
                        # Wine translates D3D into OpenGL. Without libGL a DirectX
                        # app starts and then dies with a fatal graphics error,
                        # which is a miserable thing to discover by clicking a menu entry.
                        for lib in libGL.so.1 libGLX_mesa.so.0; do
                            ldconfig -p | grep -q "$lib" || { echo "MISSING: $lib"; exit 1; }
                        done
                        # ...and specifically the 32-bit copies, since the apps are PE32.
                        # [.] not a backslash-escape: Groovy processes escapes even
                        # inside triple-single-quotes, so a regex backslash here stops
                        # the whole Jenkinsfile compiling.
                        ldconfig -p | grep 'libGL[.]so[.]1' | grep -q 'i386-linux-gnu' \
                            || { echo "MISSING: 32-bit libGL (i386) -- 32-bit apps cannot get a GL context"; exit 1; }
                        # Emulator side: retroarch plus the three cores that back the
                        # non-MAME systems. A missing core is a menu full of entries
                        # that all fail to launch, which reads as "the files are bad".
                        command -v retroarch >/dev/null || { echo "MISSING: retroarch"; exit 1; }
                        for c in nestopia gambatte genesis_plus_gx; do
                            ls /usr/lib/*/libretro/${c}_libretro.so >/dev/null 2>&1 \
                                || { echo "MISSING libretro core: $c"; exit 1; }
                            echo "  found core $c"
                        done
                        # The menu generator must emit parseable XML or every
                        # submenu silently comes up empty in openbox.
                        /usr/local/bin/retro-menu nes all | head -1 | grep -q openbox_pipe_menu \
                            || { echo "retro-menu did not emit a pipe menu"; exit 1; }
                        # NOTE: these are WARNINGS, not failures. One build reported a
                        # long-shipped script missing, and tkinter absent while the build
                        # logged "tkinter OK" -- i.e. the container inspected here was
                        # NOT the image just built (it carried the apt layer but none of
                        # the layers after it). Until that is understood, these checks
                        # cannot gate a deploy.
                        # Every script the desktop menu references. A missing one is a
                        # dead menu entry that only shows up when someone clicks it.
                        for s in launch-wine launch-scummvm launch-dosbox retro-install \
                                 retro-shell retro-audio-env retro-res launch-rom retro-menu \
                                 retro-fav retro-emu-settings retro-systems launch-mame \
                                 retro-launcher; do
                            test -e "/usr/local/bin/$s" || echo "WARNING: script not in THIS container: $s"
                        done
                        echo "smoke test OK"
                    '
                '''
            }
        }

        stage('Push to ECR') {
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'ecr-credentials', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']
                ]) {
                    sh '''
                        docker run --rm \
                            -e AWS_ACCESS_KEY_ID \
                            -e AWS_SECRET_ACCESS_KEY \
                            amazon/aws-cli ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                        docker push ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}
                        docker push ${ECR_REGISTRY}/${ECR_REPO}:latest
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker rmi ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG} || true'
        }
    }
}
