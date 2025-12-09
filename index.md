```
vars:
  user_directories:
    - { path: '/home/alice/data', owner: 'alice', mode: '0755' }
    - { path: '/home/alice/logs', owner: 'alice', mode: '0700' }
    - { path: '/home/bob/bin',    owner: 'bob',   mode: '0755' }

tasks:
  - name: Create user directories
    ansible.builtin.file:
      path: "{{ item.path }}"
      owner: "{{ item.owner }}"
      group: "{{ item.owner }}"
      mode: "{{ item.mode }}"
      state: directory
    loop: "{{ user_directories }}"

```
