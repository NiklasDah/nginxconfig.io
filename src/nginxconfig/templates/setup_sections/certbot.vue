<!--
Copyright 2020 DigitalOcean

This code is licensed under the MIT License.
You may obtain a copy of the License at
https://github.com/digitalocean/nginxconfig.io/blob/master/LICENSE or https://mit-license.org/

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and / or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions :

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.
-->

<template>
    <div>
        <ol v-if="letsEncryptActive">
            <li>
                <p>
                    {{ i18n.templates.setupSections.certbot.commentOutSslDirectivesInConfiguration }}
                    <br />
                </p>
                <BashPrism :key="sitesAvailable"
                           :cmd="`sed -i -r 's/(listen .*443)/\\1;#/g; s/(ssl_(certificate|certificate_key|trusted_certificate) )/#;#\\1/g' ${sitesAvailable}`"
                ></BashPrism>
            </li>

            <li>
                <p>
                    {{ i18n.templates.setupSections.certbot.reloadYourNginxServer }}
                    <br />
                </p>
                <BashPrism cmd="sudo nginx -t && sudo systemctl reload nginx"></BashPrism>
            </li>

            <li>
                <p>
                    {{ i18n.templates.setupSections.certbot.obtainSslCertificatesFromLetsEncrypt }}
                    <br />
                </p>
                <BashPrism :key="certbotCmds" :cmd="certbotCmds"></BashPrism>
            </li>

            <li>
                <p>
                    {{ i18n.templates.setupSections.certbot.uncommentSslDirectivesInConfiguration }}
                    <br />
                </p>
                <BashPrism :key="sitesAvailable" :cmd="`sed -i -r 's/#?;#//g' ${sitesAvailable}`"></BashPrism>
            </li>

            <li>
                <p>
                    {{ i18n.templates.setupSections.certbot.reloadYourNginxServer }}
                    <br />
                </p>
                <BashPrism cmd="sudo nginx -t && sudo systemctl reload nginx"></BashPrism>
            </li>

            <li>
                <p>
                    {{ i18n.templates.setupSections.certbot.configureCertbotToReloadNginxOnCertificateRenewal }}
                    <br />
                </p>
                <BashPrism cmd="echo -e '#!/bin/bash\nnginx -t && systemctl reload nginx' | sudo tee /etc/letsencrypt/renewal-hooks/post/nginx-reload.sh"></BashPrism>
                <BashPrism cmd="sudo chmod a+x /etc/letsencrypt/renewal-hooks/post/nginx-reload.sh"></BashPrism>
            </li>
        </ol>

        <div v-else class="field is-horizontal">
            <div class="field-body">
                <div class="field">
                    <div class="control">
                        <label class="text">
                            {{ i18n.templates.setupSections.certbot.certbotDoesNotNeedToBeSetupForYourConfiguration }}
                        </label>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
    import i18n from '../../i18n';
    import BashPrism from '../prism/bash';

    export default {
        name: 'SetupCertbot',
        components: {
            BashPrism,
        },
        display: i18n.templates.setupSections.certbot.certbot,
        key: 'certbot',
        props: {
            data: Object,
        },
        data() {
            return {
                i18n,
            };
        },
        computed: {
            letsEncryptDir() {
                return this.$props.data.global.https.letsEncryptRoot.computed.replace(/\/+$/, '');
            },
            letsEncryptActive() {
                for (const domain of this.$props.data.domains) {
                    if (domain && domain.https.certType.computed === 'letsEncrypt') {
                        return true;
                    }
                }
                return false;
            },
            sitesAvailable() {
                if (!this.$props.data.global.tools.modularizedStructure.computed)
                    return `${this.$props.data.global.nginx.nginxConfigDirectory.computed}/nginx.conf`;

                const enabledAvailable = this.$props.data.global.tools.symlinkVhost.computed ? 'available' : 'enabled';
                return this.$props.data.domains
                    .filter(domain => domain.https.certType.computed === 'letsEncrypt')
                    .map(domain => `${this.$props.data.global.nginx.nginxConfigDirectory.computed}/sites-${enabledAvailable}/${domain.server.domain.computed}.conf`)
                    .join(' ');
            },
            certbotCmds() {
                return this.$props.data.domains
                    .filter(domain => domain.https.certType.computed === 'letsEncrypt')
                    .map(domain => (
                        [
                            'certbot certonly --webroot',
                            `-d ${domain.server.domain.computed}`,
                            domain.server.wwwSubdomain.computed ? `-d www.${domain.server.domain.computed}` : null,
                            domain.server.cdnSubdomain.computed ? `-d cdn.${domain.server.domain.computed}` : null,
                            `--email ${domain.https.letsEncryptEmail.computed}`,
                            `-w ${this.letsEncryptDir}`,
                            '-n --agree-tos --force-renewal',
                        ].filter(x => x !== null).join(' ')
                    )).join('\n');
            },
        },
    };
</script>
