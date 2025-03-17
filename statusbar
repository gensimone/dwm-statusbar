#!/bin/python

import logging
import os
import json
import sys
from dataclasses import dataclass
from typing import List, Optional, Dict, Set
from asyncio import create_subprocess_shell, run
from asyncio.tasks import gather, sleep
from asyncio.subprocess import PIPE


logger = logging.getLogger(__name__)


@dataclass
class Module:
    path: str
    args: str = ''
    time: float = 0
    value: Optional[str] = None

    def __hash__(self):
        return hash((self.path, self.args, self.time))

    async def exec(self) -> None:
        proc = await create_subprocess_shell(
            cmd=self.path,
            stdout=PIPE,
            stderr=PIPE
        )

        stdout, stderr = await proc.communicate()
        if proc.returncode:
            logger.error(stderr.decode().strip())
        else:
            self.value = stdout.decode().strip()
            logger.info(self.value)

class StatusBar:
    def __init__(self, modules: List[Module], delimiter: Optional[str] = "|") -> None:
        self.modules = modules
        self.delimiter = delimiter

    async def display(self) -> None:
        str_status_bar = self.render()
        proc = await create_subprocess_shell(
            cmd=f'xsetroot -name "{str_status_bar}"',
            stdout=PIPE,
            stderr=PIPE
        )

        _, stderr = await proc.communicate()
        if proc.returncode:
            logger.error(stderr.decode())

    def render(self) -> str:
        str_status_bar = ''
        delimiter = '' if self.delimiter is None else str(self.delimiter)
        for module in self.modules:
            if module.value:
                str_status_bar += delimiter
                str_status_bar += module.value
        return str_status_bar

def load_modules(raw_modules: List[Dict]) -> List[Module]:
    modules: List[Module] = []
    for rw in raw_modules:

        if not 'path' in rw:
            raise KeyError('Missing "path" key')
        elif not isinstance(rw['path'], str):
            raise ValueError('Key "path" must be a string')
        else:
            path = rw['path']
            # if not path:
            #     raise FileNotFoundError(f'Missing path')
            if not os.path.dirname(path):
                path = f'{get_pdir()}/modules/{path}'
            if not os.path.exists(path):
                raise FileNotFoundError(f'File not found: {path}')
            if not os.path.isfile(path) or not os.access(path, os.X_OK):
                raise ValueError(f'path must be an executable file')

        module = Module(path=path)
        if 'time' in rw:
            module.time = float(rw['time'])
        if 'args' in rw:
            module.args = str(rw['args'])
        modules.append(module)
        logger.debug(f'Loaded {module}')
    return modules

def get_user_config() -> Optional[str]:
    """
    Obtains the configuration file by looking in these location:
    - $DWM_STATUS_BARList
    - $XDG_CONFIG_HOME/dwm-status-bar/config.json
    - $HOME/.dwm-status-bar.json
    """
    dwm_status_bar = os.environ.get('DWM_STATUS_BAR')
    if dwm_status_bar and os.path.exists(dwm_status_bar):
        return dwm_status_bar

    xdg_config_home = os.environ.get('XDG_CONFIG_HOME')
    if xdg_config_home:
        config = f'{xdg_config_home}/dwm-status-bar/config.json'
        if os.path.exists(config):
            return config

    home = os.environ.get('HOME')
    if home:
        config = f'{home}/.dwm-status-bar.json'
        if os.path.exists(config):
            return config
    return None

async def exec_group_once(group: List[Module], status_bar: StatusBar) -> None:
    await gather(*[x.exec() for x in group])
    await status_bar.display()

async def exec_group(time: float, group: List[Module], status_bar: StatusBar) -> None:
    await gather(*[x.exec() for x in group])
    await status_bar.display()
    await sleep(time)
    await exec_group(time, group, status_bar)

def group_modules(modules: List[Module]) -> Dict[float, List[Module]]:
    groups: Dict[float, List[Module]] = {}
    for module in modules:
        if module.time in groups:
            groups[module.time].append(module)
        else:
            groups[module.time] = [module]
    return groups

def get_pdir() -> str:
    return os.path.dirname(os.path.abspath(__file__))

def load_config_file(path: str) -> Dict:
    with open(path, mode='r') as f:
        return json.load(f)

async def main() -> None:
    format=(
        "%(name)s:"
        "%(levelname)s:"
        # "%(taskName)s:"
        "%(message)s"
    )
    logging.basicConfig(level=logging.DEBUG, format=format)

    user_config = get_user_config()
    config = None
    if user_config:
        try:
            config = load_config_file(user_config)
        except (FileNotFoundError, json.JSONDecodeError) as e:
            logger.error(e)
    if not config or not user_config:
        try:
            config = load_config_file(f'{get_pdir()}/config.json')
        except (FileNotFoundError, json.JSONDecodeError) as e:
            logger.critical(e)
            sys.exit(1)

    if 'modules' in config:
        modules = load_modules(config['modules'])
    else:
        logger.error('Missing module section in config file')
        sys.exit(1)

    status_bar = StatusBar(modules, delimiter=config.get('delimiter'))
    gm = group_modules(modules)
    await gather(*[exec_group(time, modules, status_bar) for time, modules in gm.items()])


if __name__ == "__main__":
    run(main())
