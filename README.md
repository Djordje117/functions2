# functions2

import os
import os.path
import re
from pymatgen.io.vasp.outputs import Vasprun
from pymatgen.core.structure import Element
from pymatgen.core.structure import Structure
from pymatgen.symmetry.analyzer import SpacegroupAnalyzer
import operator
from mpinterfaces.utils import add_vacuum_padding
from pymatgen.io.vasp.inputs import Incar
from decimal import Decimal
from pymatgen.io.vasp.inputs import Poscar
from pymatgen.io.vasp.inputs import Potcar
from pymatgen.matproj.rest import MPRester
from pymatgen.io.vasp.inputs import Kpoints
from openpyxl import Workbook
from openpyxl.chart import (ScatterChart,Reference,Series)

pseudos = {'H':'H','He':'He','Li':'Li_sv','Be':'Be','B':'B','C':'C','N':'N',
'O':'O','F':'F','Ne':'Ne','Na':'Na_pv','Mg':'Mg','Al':'Al','Si':'Si','P':'P','S':'S'
,'Cl':'Cl','Ar':'Ar','K':'K_sv','Ca':'Ca_sv','Sc':'Sc_sv','Ti':'Ti_sv','V':'V_sv'
,'Cr':'Cr_pv','Mn':'Mn_pv','Fe':'Fe','Co':'Co','Ni':'Ni','Cu':'Cu','Zn':'Zn','Ga':'Ga_d','Ge':'Ge_d',
'As':'As','Se':'Se','Br':'Br','Kr':'Kr','Rb':'Rb_sv','Sr':'Sr_sv','Y':'Y_sv','Zr':'Zr_sv','Nb':'Nb_sv','Mo':'Mo_sv',
'Tc':'Tc_pv','Ru':'Ru_pv','Rh':'Rh_pv','Pd':'Pd','Ag':'Ag','Cd':'Cd','In':'In_d','Sn':'Sn_d','Sb':'Sb','Te':'Te','I':'I',
'Xe':'Xe','Cs':'Cs_sv','Ba':'Ba_sv', 'La':'La','Ce':'Ce','Pr':'Pr_3','Nd':'Nd_3','Pm':'Pm_3','Sm':'Sm_3','Eu':'Eu_2','Gd':'Gd_3','Tb':'Tb_3',
'Dy':'Dy_3','Ho':'Ho_3','Er':'Er_3','Tm':'Tm_3','Yb':'Yb_2','Lu':'Lu_3','Hf':'Hf_pv','Ta':'Ta_pv','W':'W_pv','Re':'Re'
,'Os':'Os','Ir':'Ir','Pt':'Pt','Au':'Au','Hg':'Hg','Tl':'Tl_d','Pb':'Pb_d','Bi':'Bi_d','Po':'Po_d','At':'At_d','Rn':'Rn','Fr':'Fr_sv'
,'Ra':'Ra_sv','Ac':'Ac','Th':'Th','Pa':'Pa','U':'U','Np':'Np','Pu':'Pu','Am':'Am','Cm':'Cm'}

def copier(origin,target):
    A = os.getcwd()
    LIST = ['KPOINTS','INCAR','POSCAR','POTCAR','WAVECAR','OSZICAR','OUTCAR','IBZKPT']
    for child in os.listdir(origin):
        child_origin = os.path.join(origin,child)
        if os.path.isdir(child_origin):
            os.chdir(target)
            if not os.path.exists(child):
                os.mkdir(child)
            child_target = os.path.join(target,child)
            os.chdir(child_origin)
            for i in range(len(LIST)):
                if os.path.exists(LIST[i]):
                    string = 'cp '+child_origin+'/'+LIST[i]+' '+child_target
                    print string
    os.chdir(A)
    
def print_dic(DIC = {}):
    for key,value in DIC.iteritems():
        print key,value
        
def create_energy_dic(DIR,extension = ''):
    A = os.getcwd()
    DIC = {}
    for child in os.listdir(DIR):
        path = os.path.join(DIR,child,extension)
        if os.path.isdir(path):
            os.chdir(path)
            if check_if_done() and proper():
                DIC.update({child: extract_energy()})
    os.chdir(A)
    return DIC                        
                    
def get_name():
    if os.path.exists('POSCAR'):
        POS = open('POSCAR','r').readlines()
        name = POS[5].split()
        num = POS[6].split()
        if int(num[1]) > int(num[0]):
            Name = name[0]+num[0]+name[1]+num[1]
        else:
            Name = name[1]+num[1]+name[0]+num[0]
        return Name

def formation_energy(twoD_DIR,threeD_DIR):
    twoList = []
    threeList = []
    Titel = 'MPs 2D_energy 3D_energy formation_energy\n'
    form = open('formation_energies','w')
    form.write(Titel)
    
    os.chdir(twoD_DIR)
    two_info = open('energy_info','r').readlines()
    for i in range(len(two_info)):
        twoList.append(two_info[i].split())
        
    os.chdir(threeD_DIR)
    three_info = open('energy_info','r').readlines()        
    for i in range(len(three_info)):
        threeList.append(three_info[i].split()) 
               
    for i in range(len(twoList)):
        for j in range(len(threeList)):
            if twoList[i][0] == threeList[j][0]:
                two_energy = float(twoList[i][1])
                three_energy = float(threeList[i][1])
                two_num = float(num(os.path.join(twoD_DIR,twoList[i][0])))
                three_num = float(num(os.path.join(threeD_DIR,threeList[i][0])))
                form_energy = (two_energy/two_num-three_energy/three_num)*1000
                form.write(twoList[i][0]+'  '+twoList[i][1]+'   '+threeList[j][1]+'  '+str(form_energy)+'\n')
    form.close()
    
        
def check_band():
    if os.path.exists('OSZICAR'):
        OSZ = open('OSZICAR','r').readlines()
        if OSZ and  'E0= ' in OSZ[len(OSZ)-1]:
            return 1
        else: return 0
    else: return 0

def num(DIR):
    A = os.getcwd()
    os.chdir(DIR)
    if os.path.exists('POSCAR'):
        sume= 0
        line = open('POSCAR','r').readlines()
        numb = re.findall('-?[0-9]+', line[6])
        for i in range(len(numb)):
            sume +=int(numb[i])
        os.chdir(A)
        return sume
    else:
        os.chdir(A)
        raise ValueError('NO POSCAR found')
        

def get_magmom_string_anti_fero():
    POS = open('POSCAR','r').readlines()
    Name = POS[5].split()
    Num = POS[6].split()
    string = ''
    if Element(Name[0]).is_transition_metal:
        a= int(Num[0])
        while a > 0:
            string += '5 -5 '
            a -=2
        string += Num[1]+'*0.5'
    else:
        string += Num[0]+'*0.5'
        a = int(Num[1])
        while a > 0:
            string += ' 5 -5'
            a -=2
    return string
    
def check_if_done():
    if os.path.exists('OUTCAR'):
            OUT = open('OUTCAR','r').readlines()
            l =len(OUT)
            for i in xrange(l-1,int(90*l/100),-1):
                    if 'reached required accuracy' in OUT[i]:
                                return True
            return 0
    return 0
    
def proper():
    if os.path.exists('OSZICAR'):
        OSZ = open('OSZICAR','r').readlines()
        if len(OSZ) < 3:
            print 'OSZICAR empty'
            return 0
        numb = re.findall('^ *[0-9]+', OSZ[len(OSZ)-1])
        number_of_ionic_steps = int(re.sub(' *','',numb[0]))
        if number_of_ionic_steps > 5:
            return 0
        else: return 1
    else:
        return 0
        
def check_CONTCAR():
    if os.path.exists('CONTCAR'):
        CO = open('CONTCAR','r').read()
        if CO:
            os.system('mv CONTCAR POSCAR')
            return 1
        else: return 0
    else: return 0
    
def extract_energy():
    OSZ = open('OSZICAR','r').readlines()
    sake = re.sub('.*E0= ','',OSZ[len(OSZ)-1])
    sake = re.sub(' *d .*\n','',sake)
    sake = float(sake)
    return sake

def energy_parser():
    info = open('energy_info','r').readlines()
    Break = 2 #initialization to non-zero number
    options = 'Total energy = 1, mp-id = 2, EXIT = 0'
    while Break != 0:
        print (options)
        choice = int(raw_input('Enter which data you want from magnet_info: '))
        if choice == 0:
            break
        if choice > 2 or choice <0:
            print '\nNumber out of range, try again.'
            continue
        if choice == 1:
            File = open('energy','w')
            for i in range(len(info)):
                Data = info[i].split()
                File.write(Data[1]+'\n')
            File.close()
        else:
            File = open('MP-IDs','w')
            for i in range(len(info)):
                Data = info[i].split()
                File.write(Data[0]+'\n')
            File.close()
            
def symmetrize():
    structure = Structure.from_file('POSCAR')
    symm = SpacegroupAnalyzer(structure).get_refined_structure()
    prim = SpacegroupAnalyzer(symm).find_primitive()
    prim.to(fmt="poscar", filename="POSCAR_symm")
    
def nice_format(File,Titel_present = False):
    if os.path.exists(File):
        content = open(File,'r').readlines()
        if not Titel_present:
            new_file = open(File,'w')
            for i in range(len(content)):
                string = ''
                temp = content[i].split()
                for i in range(len(temp)):
                    if len(temp[i]) >12:
                         string +=temp[i]+(20 - len(temp[i]))*' '
                    else:
                        string +=temp[i]+(15 - len(temp[i]))*' '
                string +='\n'
                new_file.write(string)
            new_file.close()
        
        else:
            titel = content[0]
            new_file = open(File,'w')
            new_file.write(titel)
            for i in range(1,len(content)):
                string = ''
                temp = content[i].split()
                for i in range(len(temp)):
                    if len(temp[i]) >12:
                         string +=temp[i]+(20 - len(temp[i]))*' '
                    else:
                        string +=temp[i]+(15 - len(temp[i]))*' '
                string +='\n'
                new_file.write(string)
            new_file.close()
    else: raise IOError(File+' does not exist')

def check_if_HSE_done():
    if os.path.exists('OUTCAR'):
            OUT = open('OUTCAR','r').readlines()
            l =len(OUT)
            for i in xrange(l-1,int(98*l/100),-1):
                    if 'General timing and accounting informations for this job:' in OUT[i]:
                                return True
            return 0
    return 0
    
def create_energy_dic_anti_fero_magnetism(DIR):
    A = os.getcwd()
    DIC = {}
    for child in os.listdir(DIR):
        path = os.path.join(DIR,child,'anti_fero_magnetism')
        if os.path.isdir(path):
            os.chdir(path)
            if check_if_done() and proper():
                DIC.update({child: extract_energy()})
    os.chdir(A)
    return DIC

def formation_energy_anti_fero_magnetism(twoD_DIR,threeD_DIR):
    twoList = []
    threeList = []

    os.chdir(twoD_DIR)
    Titel = 'MPs 2D_energy 3D_energy formation_energy\n'
    form = open('formation_energies_anti_fero_magnetism','w')
    form.write(Titel)
    two_info = open('energy_info_anti_fero_magnetism','r').readlines()
    for i in range(len(two_info)):
        twoList.append(two_info[i].split())

    os.chdir(threeD_DIR)
    three_info = open('energy_info_anti_fero_magnetism','r').readlines()
    for i in range(len(three_info)):
        threeList.append(three_info[i].split())

    for i in range(len(twoList)):
        for j in range(len(threeList)):
            if twoList[i][0] == threeList[j][0]:
                form_energy = (float(twoList[i][1])-float(threeList[j][1]))/num(os.path.join(twoD_DIR,twoList[i][0]))*1000
                form.write(twoList[i][0]+'  '+twoList[i][1]+'   '+threeList[j][1]+'  '+str(form_energy)+'\n')
    form.close()
    
def create_energy_dic_magnetism(DIR):
    DIC = {}
    for child in os.listdir(DIR):
        path = os.path.join(DIR,child,'magnetism')
        if os.path.isdir(path):
            os.chdir(path)
            if check_if_done() and proper():
                DIC.update({child: extract_energy()})
    return DIC
    
def formation_energy_magnetism(twoD_DIR,threeD_DIR):
    twoList = []
    threeList = []

    os.chdir(twoD_DIR)
    Titel = 'MPs 2D_energy 3D_energy formation_energy\n'
    form = open('formation_energies_magnetism','w')
    form.write(Titel)
    two_info = open('energy_info_magnetism','r').readlines()
    for i in range(len(two_info)):
        twoList.append(two_info[i].split())

    os.chdir(threeD_DIR)
    three_info = open('energy_info_magnetism','r').readlines()
    for i in range(len(three_info)):
        threeList.append(three_info[i].split())

    for i in range(len(twoList)):
        for j in range(len(threeList)):
            if twoList[i][0] == threeList[j][0]:
                form_energy = (float(twoList[i][1])-float(threeList[j][1]))/num(os.path.join(twoD_DIR,twoList[i][0]))*1000
                form.write(twoList[i][0]+'  '+twoList[i][1]+'   '+threeList[j][1]+'  '+str(form_energy)+'\n')
    form.close()

def write_energy_from_dic(choice,DIC = {}):
    if choice == 0:
        raise SystemExit

    if choice ==1:
        energy = open('energy_info','w')
    if choice ==2:
        energy = open('energy_info_magnetism','w')
    if choice ==3:
        energy = open('energy_info_anti_fero_magnetism','w')
    if choice ==4:
        energy = open('energy_info_spin_x','w')
        
    if choice ==5:
        energy = open('energy_info_spin_z','w')
        
    for key,value in DIC.iteritems():
        energy.write(key+'  '+str(value)+'\n')
    energy.close()
    
def extract_magnetisation():
    if check_if_done() and proper():
        OUT = open('OUTCAR','r').readlines()
        l = len(OUT)
        for i in xrange(l-1,0,-1):
            if ' magnetization (x)' in OUT[i]:
                break
        for j in xrange(i,l-1):
            if 'tot  ' in OUT[j]:
                total_magnetization = OUT[j].split()
                break
        print '  '.join(total_magnetization)
  
def duplicate_job_delete(ID_file):
    if os.path.exists(ID_file):
        jobs = open(ID_file,'r').readlines()
    else:
        raise IOError('No '+ID_file+' file found.')
    List = []
    redundent = []
    for k in range(len(jobs)):
        List.append(jobs[k].split())
        
    for i in range(len(List)-1):
        for j in xrange(i+1,len(List)):
            if List[i][1] == List[j][1]:
                if List[i][2] == 'R' and List[j][2] == 'Q':
                    redundent.append(List[j][0])
                    break
                if List[j][2] == 'R' and List[i][2] == 'Q':
                    redundent.append(List[i][0])
                    break
                if List[j][2] == 'R' and List[i][2] == 'R':
                    redundent.append(List[i][0])
                    redundent.append(List[j][0])
                    break
                if List[j][2] == 'Q' and List[i][2] == 'Q':
                    redundent.append(List[j][0])
                    break
    for k in range(len(redundent)):
        os.system('qdel '+redundent[k])
    
def name_substitute(substitute):
    if os.path.exists('submit'):
        SUB = open('submit','r').read()
        SUB = re.sub('#PBS -N.*\n','#PBS -N '+substitute+'\n',SUB)
        New_sub = open('submit','w')
        New_sub.write(SUB)
        New_sub.close()
    else:
        raise IOError('No submit file found')
        
def unique_name(name,List = []):
    for i in range(len(List)):
        if List[i] == name:
            print 'same '+name+' in ' +os.getcwd()
            return List
    List.append(name)
    return List
    
def CONTCAR_proper_2D():
    LIST = []
    try:
        CON = open('CONTCAR','r').readlines()
        if not CON:
            print 'CONTCAR is empty'
            return False
        LIST.append(CON[2].split())
        LIST.append(CON[3].split())
        LIST.append(CON[4].split())
        for i in range(len(LIST)):
            for j in range(2):
                if abs(float(LIST[i][j])) > 10.0:
                    return False
                    
        for i in range(len(LIST)):
            if abs(float(LIST[i][2])) > 42.0:
                return False
        return True
    except:
        CUR_DIR = os.getcwd().split('/')
        print ('NO CONTCAR present in '+CUR_DIR[len(CUR_DIR)-1])
        try:
            CON = open('POSCAR','r').readlines()
            if not CON:
                print 'POSCAR is empty'
                return False
            LIST.append(CON[2].split())
            LIST.append(CON[3].split())
            LIST.append(CON[4].split())
            for i in range(len(LIST)):
                for j in range(2):
                    if abs(float(LIST[i][j])) > 10.0:
                        return False
                    
            for i in range(len(LIST)):
                if abs(float(LIST[i][2])) > 40.0:
                    return False
            return True
        except:
            CUR_DIR = os.getcwd().split('/')
            print ('NO POSCAR present in '+CUR_DIR[len(CUR_DIR)-1])
            
def CONTCAR_proper_3D():
    LIST = []
    try:
        CON = open('CONTCAR','r').readlines()
        if not CON:
            print 'CONTCAR is empty'
            return False
        LIST.append(CON[2].split())
        LIST.append(CON[3].split())
        LIST.append(CON[4].split())
        for i in range(len(LIST)):
            for j in range(2):
                if abs(float(LIST[i][j])) > 10.0:
                    return False
                    
        for i in range(len(LIST)):
            if abs(float(LIST[i][2])) > 25.0:
                return False
        return True
    except:
        CUR_DIR = os.getcwd().split('/')
        print ('NO CONTCAR present in '+CUR_DIR[len(CUR_DIR)-1])
        try:
            CON = open('POSCAR','r').readlines()
            if not CON:
                print 'POSCAR is empty'
                return False
            LIST.append(CON[2].split())
            LIST.append(CON[3].split())
            LIST.append(CON[4].split())
            for i in range(len(LIST)):
                for j in range(2):
                    if abs(float(LIST[i][j])) > 10.0:
                        return False
                    
            for i in range(len(LIST)):
                if abs(float(LIST[i][2])) > 25.0:
                    return False
            return True
        except:
            CUR_DIR = os.getcwd().split('/')
            print ('NO POSCAR present in '+CUR_DIR[len(CUR_DIR)-1])

def proper_CONTCAR_all_directories(DIR,extension=''):
    LIST = []
    for child in os.listdir(DIR):
        path = os.path.join(DIR,child,extension)
        if os.path.isdir(path):
            os.chdir(path)
            if os.path.exists('CONTCAR'):
                POS = open('CONTCAR','r').readlines()
                LIST.append(POS[2].split())
                LIST.append(POS[3].split())
                for i in range(len(LIST)):
                    for j in range(3):
                        if abs(float(LIST[i][j])) > 10:
                            print child + ' CONTCAR not proper'
                            break
        else:
            POS = open('POSCAR','r').readlines()
            LIST.append(POS[2].split())
            LIST.append(POS[3].split())
            for i in range(len(LIST)):
                for j in range(3):
                    if abs(float(LIST[i][j])) > 10:
                        print child+' POSCAR not proper'
                        break
            LIST = []
            
def energy_comperator():
    LIST, LIST_mag, LIST_ant = [],[],[]
    comp = open('compared_energies','w')
    form = open('formation_energies','r').readlines()
    form_mag = open('formation_energies_magnetism','r').readlines()
    form_ant = open('formation_energies_anti_fero_magnetism','r').readlines()
    
    for i in range(1,len(form)):
        LIST.append(form[i].split())
        
    for i in range(1,len(form_mag)):
        LIST_mag.append(form_mag[i].split())
        
    for i in range(1,len(form_ant)):
        LIST_ant.append(form_ant[i].split())
        
    for i in range(len(LIST)):
        for j in range(len(LIST_mag)):
            for k in range(len(LIST_ant)):
                if LIST[i][0] == LIST_mag[j][0] and LIST[i][0] == LIST_ant[k][0]:
                    
                    index, value = min(enumerate([float(LIST[i][1]),float(LIST_mag[j][1]), float(LIST_ant[k][1])]), key=operator.itemgetter(1))
                    
                    if index == 0:
                        sub_str = 'relax'
                    elif index ==1:
                        sub_str = 'magne'
                    else:
                        sub_str = 'anti_'
                    
                    string = LIST[i][0] + ' ' +str(value) + ' ' + sub_str+' '
                    
                    index, value = min(enumerate([float(LIST[i][2]),float(LIST_mag[j][2]), float(LIST_ant[k][2])]), key=operator.itemgetter(1))
                    
                    if index == 0:
                        sub_str1 = 'relax'
                    elif index ==1:
                        sub_str1 = 'magne'
                    else:
                        sub_str1 = 'anti_'
                    
                    string+= str(value)+ ' ' + sub_str1 + ' '
                    
                    
                    index, value = min(enumerate([float(LIST[i][3]),float(LIST_mag[j][3]), float(LIST_ant[k][3])]), key=operator.itemgetter(1))
                    
                    if index == 0:
                        sub_str2 = 'relax'
                    elif index ==1:
                        sub_str2 = 'magne'
                    else:
                        sub_str2 = 'anti_'
                    
                    string+= str(value)+ ' ' + sub_str2 + '\n'
                    comp.write(string)
                    break
    comp.close()
    
def job_id():
    Dir = open('job_id_active.txt','w')
    print 'Job IDs are writen in job_id_active.txt'
    os.system('qstat -a >> jobs.txt')
    count = 0
    if os.path.exists('jobs.txt'):
        jobs= open('jobs.txt','r').readlines()
        for i in range(1,len(jobs)):
            re.sub(' +',' ',jobs[i])
            lis = jobs[i].split()
            if len(lis)<4:
                continue
            if 'djordje'in lis[1]:
                print lis
                A = lis[0]+' '+lis[3]+'  '+lis[9]+'\n'
                Dir.write(A)
                count +=1
    Dir.close()
    os.system('rm jobs.txt')
    print ('number of jobs is %d'%count)

def write_active_job(DIR):
    if os.path.exists('job_id_active.txt'):
        Dir = open('job_id_active.txt','r').readlines()
        File = open('Active_directories','w')

        for child in os.listdir(DIR):
            test_path = os.path.join(DIR, child)
            if os.path.isdir(test_path):
                os.chdir(test_path)
                for i in range(len(Dir)):
                    A = Dir[i].split()
                    if os.path.exists('submit'):
                        submit = open('submit','r').read()
                        if A[1] in submit:
                             print child
                             string = child + '\n'
                             File.write(string)
                             break
    File.close()
def check_if_active(DIR,File):
    os.chdir(DIR)
    if os.path.exists('Active_directories'):
        AC = open('Active_directories','r').read()
        if File[0:20] in AC:
            return 1
        else:
            return 0
            
def new_job_id():
    os.system('qstat -u djordje.gluhovic > ID.txt')
    c=0
    ID = open('ID.txt','r').readlines()
    text = open('job_id_active.txt','w')
    for i in range(5,len(ID)):
        lis = ID[i].split()
        print lis
        A = lis[0]+' '+lis[3]+'  '+lis[9]+'\n'
        text.write(A)
        c+=1
    text.close()
    os.system('rm ID.txt')
    print ('number of jobs is %d'%c)
    
def find_line(word,FILE):#Used to check if 2D and 3D have the same POTCAR, by searchig for word 'TITEL'
    LIST = []
    with open(FILE) as search:
            for line in search:
                    line = line.rstrip()  # remove '\n' at end of line
                    if word in line:
                        LIST.append(line)
    return LIST
    
def copy_DIR(origin,target):
    LIST = ['KPOINTS','INCAR','POSCAR','POTCAR','WAVECAR','OSZICAR','OUTCAR','CHGCAR']
    if os.path.isdir(origin) and os.path.isdir(target):
        for i in range(len(LIST)):
            os.system('cp '+origin+'/'+LIST[i]+' '+target+'/'+LIST[i])
    else:
        raise ValueError('Origin or targer is not a directory')
    
def add_vacuum(vacuum = 20):
    struct_2d = Structure.from_file('POSCAR')
    new_struct = add_vacuum_padding(struct_2d, vacuum)
    new_struct.to(filename='POSCAR', fmt='poscar')
    
def backup_DIR(origin,target):
    if os.path.isdir(origin) and os.path.isdir(target):
        name = origin.split('/')
        os.chdir(target)
        if not os.path.exists(name[len(name)-1]):
            os.mkdir(name[len(name)-1])
        path = os.path.join(target,name[len(name)-1])
        LIST = ['KPOINTS','INCAR','POSCAR','POTCAR','WAVECAR','OSZICAR','OUTCAR','CHGCAR']
    
        for i in range(len(LIST)):
            if not os.path.isdir(os.path.join(origin,LIST[i])):
                os.system('cp '+origin+'/'+LIST[i]+' '+path+'/'+LIST[i])
            else:
                print LIST[i]
    else:
        raise ValueError('Origin or targer is not a directory')
        
def CONTCAR_change(tolerance = 0.05):
    if os.path.exists('CONTCAR') and os.path.exists('POSCAR'):
        LIST_CON, LIST_POS = [],[]
        CON = open('CONTCAR','r').readlines()
        if not CON:
            print 'CONTCAR is empty.'
            return False
        POS = open('POSCAR','r').readlines()
        
        for i in range(2,5):
            LIST_CON.append(CON[i].split())
            LIST_POS.append(POS[i].split())
        for i in range(3):
            for j in range(3):
                LIST_POS[i][j] = float(LIST_POS[i][j])
                LIST_CON[i][j] = float(LIST_CON[i][j])
                if abs((LIST_POS[i][j]-LIST_CON[i][j])/LIST_POS[i][j]) > tolerance:
                    print 'One of the base vectors increased more than allowed tollerance'
                    return
        return True
    else:
        print 'CONTCAR or POSCAR not present in the direcory'
        return False
        
        
def compare_magnetization(DIR, tollerance = 0.1):
    twoD = '2D'
    threeD = '3D'
    MAG2, MAG3 = [], []
    temp = []
    os.chdir(os.path.join(DIR,twoD))
    mag_file2 = open('magnet_info','r').readlines()
    
    os.chdir(os.path.join(DIR,threeD))
    mag_file3 = open('magnet_info','r').readlines()
    
    for i in range(1,len(mag_file2)):
        A = mag_file2[i].split()
        temp.append(A[0].split('_')[0])
        temp.append(A[5])
        MAG2.append(temp)
        temp = []
       
    for i in range(1,len(mag_file3)):
        A = mag_file3[i].split()
        temp.append(A[0].split('_')[0])
        temp.append(A[5])
        MAG3.append(temp)
        temp = []
        
    for i in range(len(MAG2)):
        for j in range(len(MAG3)):
            if MAG2[i][0] == MAG3[j][0]:
                diff = float(MAG2[i][1]) - float(MAG3[j][1])
                if abs(diff) > tollerance:
                    print 'Magnetization changed from ' +MAG3[j][1]+' to' + MAG2[i][1]
                    
                    
def delete_all_jobs(only_active = False):
    new_job_id()
    File = open('job_id_active.txt','r').readlines()
    if not only_active:
        for i in range(len(File)):
            os.system('qdel '+ File[i].split()[0])
    else:
       for i in range(len(File)):
           status = File[i].split()[2]
           if status !='R':
               os.system('qdel '+ File[i].split()[0])   
            
           
def round_numbers(File, tolerance = 0.1, columns = [], start_line = 0, end = 0):
    try:
        FILE = open(File,'r').readlines()
    except IOError:
        raise IOError('No '+File +' found')
    List = []
    temp = []
    if end == 0:
        end = len(FILE)
    for i in range(start_line,end):
        A = FILE[i].split()
        for k in range(len(A)):
            temp.append(A[k])
        for j in range(len(columns)):
            temp.append('.')
        List.append(temp)
        temp =[]
    count = len(FILE[start_line].split()) #number of columns of the file 
    for i in range(len(columns)): #checking 
        if (int(columns[i])-1) > (count):
            raise ValueError('Some of the desired rows exceed the dimension of the file')
    
    
    sa = 0
    for j in range(len(columns)):
        for i in range(len(List)):
              if abs(int(float(List[i][columns[j]])) - float(List[i][columns[j]])) <= tolerance:
                  List[i][sa+count] = str(int(float(List[i][columns[j]])))
                  
              elif abs(int(float(List[i][columns[j]])) - float(List[i][columns[j]])+1) <= tolerance:
                  List[i][sa+count] = str(int(float(List[i][columns[j]]))+1)
              elif abs(int(float(List[i][columns[j]])) - float(List[i][columns[j]])-1) <= tolerance:
                  List[i][sa+count] = str(int(float(List[i][columns[j]]))-1)
              else:
                  List[i][sa+count] = str(round(float(List[i][columns[j]]),5))
        sa+=1 
        
    print List  
    NEW_file = open(File,'r').read()
    sa = 0 
    for j in range(len(columns)):
        for i in range(len(List)):
            NEW_file= NEW_file.replace((List[i][columns[j]]),str((List[i][sa+count])))
        sa+=1
    rounded = open(File+'_rounded','w')
    rounded.write(NEW_file)
    rounded.close()
    nice_format(File+'_rounded')
                  
def get_NBANDS_string(DIR):
    try:
        os.chdir(DIR)
    except:
        raise ValueError('No directory for get_NBANDS_string found')
    if os.path.exists('OUTCAR'):
        OUT = open('OUTCAR','r').readlines()
        for i in range(len(OUT)):
            if 'NBANDS' in OUT[i]:
                numb = re.findall('-?[0-9]+', OUT[i])
                break
        return 2*int(numb[len(numb)-1])
    else:
        raise IOError('NO OUTCAR found')
        
def get_magmom_spin(direction,yield_magnetization):
    if not os.path.exists('OSZICAR'):
        raise IOError('No OSZICAR present.')
    OSZ= open('OSZICAR','r').readlines()
    l = len(OSZ)-1
    mage = OSZ[l]
    mage = re.sub('.+mag=\s+','',mage)
    mage = re.sub('\n','',mage)
    POS = open('POSCAR','r').readlines()
    num = POS[6].split()
    if int(num[0]) < int(num[1]):
        scale = int(num[0])
        majority = int(num[1])
    else:
        scale = int(num[1])
        majority = int(num[0])
    
    magnetism = float(mage)/scale
    
    
    string = ('0 0 '+str(magnetism)+' ')*scale + ' 0 0 0'*majority
    if yield_magnetization:
        return magnetism, string
    else:
        return string
    
        
def create_spin_polarized_comp_all_directories(DIR,direction, force_overwrite = False, submit = True):
    LIST = ['KPOINTS','POSCAR','POTCAR','CONTCAR','submit']
    if 'x' in direction:
        saxis = '1 0 0'
    else:
        saxis = '0 0 1'
    
    MAG_INCAR_DICT = {'PREC': 'Accurate', 'ENCUT': 600, 'IBRION': -1,'ISPIN': 2, 'NSW ': 0, 'ISIF': 2,'ISYM': 0,'LMAXMIX':4,
     'LCHARG': False, 'ISMEAR': 1,'LWAVE': False,'NPAR': 8, 'EDIFF': 1e-8,'SIGMA': 0.1,'LSORBIT': '.True.',
    'ICHARG':11,'SAXIS': saxis}
    
    First_step_INCAR = {'PREC':'Accurate','ENCUT':450,'IBRION':2,'NSW':0,'ISIF':3,'LCHARG':'True','ISMEAR':1,'SIGMA':0.1,'EDIFF':'1E-6'
    ,'NPAR':8,'LWAVE':'True'}
   
    for child in os.listdir(DIR):
        First_phase_done = True #initialization
        path = os.path.join(DIR,child,'magnetism')
        if os.path.isdir(path):
            os.chdir(path)
            if check_if_done() and proper():
                magnetization , temp = get_magmom_spin()
                if int(magnetization) < 0.5:
                    continue
                MAG_INCAR_DICT.update({'MAGMOM': get_magmom_spin(direction,A=False)})
                MAG_INCAR_DICT.update({'NBANDS': get_NBANDS_string(path)})
                os.chdir(os.path.join(DIR,child))
                if not os.path.exists('spin_polarized'+'_'+direction):
                    os.mkdir('spin_polarized'+'_'+direction)
                    First_phase_done = False
                spin_path = os.path.join(DIR,child,'spin_polarized'+'_'+direction)
                os.chdir(spin_path)
                #Check if the first phase is done
                if os.path.exists('INCAR'):
                    INC = open('INCAR','r').read()
                    if 'LMAXMIX' not in INC:
                        if not check_if_done():
                            First_phase_done = False
                    else:
                        if check_if_done():
                            if not force_overwrite:
                                continue
                        else:
                            print child+' '+'spin in '+direction+' not done'
                else:
                    First_phase_done = False
                if not First_phase_done:
                    First_step_INCAR.update({'NBANDS':get_NBANDS_string(path)})
                    os.chdir(spin_path)
                    Incar.from_dict(First_step_INCAR).write_file('INCAR')
                else:
                    
                    Incar.from_dict(MAG_INCAR_DICT).write_file('INCAR')
                for i in range(len(LIST)):
                    string = 'cp '+ path+'/'+LIST[i] +' .'
                    os.system(string)
                if os.path.exists('CONTCAR'):
                    os.system('mv CONTCAR POSCAR')
                name_substitute(get_name()+'_2D_spin_'+direction)
                update_submit_file(['e','p'],['/home/mashton/vasp.5.4.1/bin/vasp_ncl','32'])
                if submit:
                    os.system('sbatch submit')
                    

def update_submit_file(flags = [], values = []):
    #flags: t = time, n = name, m = memory requested, p = number of precessors requested, q = queue, e = executable, @ = email
    if len(flags) != len(values):
        raise IOError('Sizes of 2 lists do not match')
    if os.path.exists('submit'):
        sub = open('submit','r').read()
    else:
        raise IOError('No submit file present')
    for i in range(len(flags)):
        if flags[i] == 't': #time
            sub = re.sub('#SBATCH -t.*\n','#SBATCH -t '+values[i]+'   #Walltime in hh:mm:ss or d-hh:mm:ss\n',sub)
            
        elif flags [i] == 'n': #name
           sub = re.sub('#PBS -N.*\n','#PBS -N '+values[i] +'\n',sub)
        
        elif flags [i] == 'm': #memory
           sub = re.sub('#SBATCH --mem.*\n','#SBATCH --mem-per-cpu='+str(values[i]).strip(' ') +'   #Per processor memory request\n',sub)
        
        elif flags [i] == 'p':#processors
           sub = re.sub('#SBATCH --ntasks=.*\n','#SBATCH --ntasks='+str(values[i]).strip(' ')+'\n',sub)
        
        elif flags [i] == 'q':#queu
           sub = re.sub('#SBATCH --qos=.*\n','#SBATCH --qos='+values[i].strip(' ')+'\n',sub)
        
        elif flags [i] == 'e': #executable
           sub = re.sub('mpiexec.*log','mpiexec '+values[i]+'  > job.log\n',sub)
       
        elif flags [i] == '@': #executable
           sub = re.sub('#SBATCH --mail-user=.*\n','#SBATCH --mail-user='+values[i]+'\n',sub)
        
        new_sub = open('submit','w')
        new_sub.write(sub)
        new_sub.close()
        
def remove_junk(DIR):
    LIST = ['AECCAR*','CHG','WAVECAR','DOSCAR','PROCAR','REPORT','PCDAT','XDATCAR','EIGENVAL','job_1*','err*']
    string = 'rm'
    for i in range(len(LIST)):
        string+= ' '+LIST[i]
    os.chdir(DIR)
    os.system(string)
    for child in os.listdir(DIR):
        path  = os.path.join(DIR,child)
        if os.path.isdir(path):
            remove_junk(path)

        
def create_spin_polarized_comp(direction, force_overwrite = False, submit = True):
    #This function should be started in ordinary relaxation directory, which containes magnetisation directory 
    LIST = ['KPOINTS','POSCAR','POTCAR','CONTCAR','submit']
    if 'x' in direction:
        saxis = '1 0 0'
    else:
        saxis = '0 0 1'
    
    MAG_INCAR_DICT = {'PREC': 'Accurate', 'ENCUT': 600, 'IBRION': -1,'ISPIN': 2, 'NSW ': 0, 'ISIF': 2,'ISYM': 0,'LMAXMIX':4,
     'LCHARG': False, 'ISMEAR': 1,'LWAVE': False,'NPAR': 4, 'EDIFF': 1e-8,'SIGMA': 0.1,'LSORBIT': '.True.',
    'ICHARG':11,'SAXIS': saxis}
    
    First_step_INCAR = {'PREC':'Accurate','ENCUT':500,'IBRION':2,'NSW':0,'ISIF':3,'LCHARG':'True','ISMEAR':1,'SIGMA':0.1,'EDIFF':'1E-6'
    ,'NPAR':4,'LWAVE':'True'}
   
    DIR = os.getcwd() # Child directory
    First_phase_done = True #initialization
    path = os.path.join(DIR,'magnetism')
    if os.path.isdir(path):
            os.chdir(path)
            if check_if_done() and proper():
                magnetization , temp = get_magmom_spin(direction,True)
                if int(magnetization) < 2:
                    return
                
                os.chdir(DIR) #create spin_polarized_X directory
                if not os.path.exists('spin_polarized'+'_'+direction):
                    os.mkdir('spin_polarized'+'_'+direction)
                    First_phase_done = False
                spin_path = os.path.join(DIR,'spin_polarized'+'_'+direction)
                os.chdir(spin_path)
                #Check if the first phase is done
                if os.path.exists('INCAR'):
                    INC = open('INCAR','r').read()
                    if 'LMAXMIX' not in INC:
                        if not check_if_done():
                            First_phase_done = False
                    else:
                        if check_if_done():
                            if not force_overwrite:
                                return
                        else:
                            print DIR+' '+'spin in '+direction+' not done'
                else:
                    First_phase_done = False
                if not First_phase_done:
                    First_step_INCAR.update({'NBANDS':get_NBANDS_string(path)})
                    os.chdir(spin_path)
                    Incar.from_dict(First_step_INCAR).write_file('INCAR')
                else:
                    MAG_INCAR_DICT.update({'NBANDS': get_NBANDS_string(path)})
                    MAG_INCAR_DICT.update({'MAGMOM': get_magmom_spin(direction,A=False)})
                    os.chdir(spin_path)
                    Incar.from_dict(MAG_INCAR_DICT).write_file('INCAR')
                for i in range(len(LIST)):
                    string = 'cp '+ path+'/'+LIST[i] +' .'
                    os.system(string)
                if os.path.exists('CONTCAR'):
                    os.system('mv CONTCAR POSCAR')
                name_substitute(get_name()+'_2D_spin_'+direction)
                update_submit_file(['e','p'],['/home/mashton/vasp.5.4.1/bin/vasp_ncl','32'])
                if submit:
                    print 'submit'
                    #os.system('sbatch submit')
    else:
        print 'No magnetism directory in '+DIR
        return
        
def makeKPOINTS(twoD, MeshType,Length, desired_directory):
        #Input Variables
        MeshType = MeshType
        TwoDimensional = twoD
        l = Length
        input_POSCAR = desired_directory+'/POSCAR'
        output_KPOINTS = desired_directory+'/KPOINTS'

        POSCAR = open(input_POSCAR, 'r')
        lines = POSCAR.readlines()
        scale = Decimal.from_float(float(lines[1].split()[0]))

        #Define original lattice vectors

        a1 = lines[2].split()
        a2 = lines[3].split()
        a3 = lines[4].split()

        a11 = scale*Decimal.from_float(float(a1[0]))
        a12 = scale*Decimal.from_float(float(a1[1]))
        a13 = scale*Decimal.from_float(float(a1[2]))

        a21 = scale*Decimal.from_float(float(a2[0]))
        a22 = scale*Decimal.from_float(float(a2[1]))
        a23 = scale*Decimal.from_float(float(a2[2]))

        a31 = scale*Decimal.from_float(float(a3[0]))
        a32 = scale*Decimal.from_float(float(a3[1]))
        a33 = scale*Decimal.from_float(float(a3[2]))

        #Calculate the determinant

        det = a11*(a22*a33-a23*a32)-a12*(a21*a33-a23*a31)+a13*(a21*a32-a22*a31)
        b11 = (a22*a33-a23*a32)/det
        b12 = (a21*a33-a23*a31)/det
        b13 = (a21*a32-a22*a31)/det

        b21 = (a12*a33-a13*a32)/det
        b22 = (a11*a33-a13*a31)/det
        b23 = (a11*a32-a12*a31)/det

        b31 = (a12*a23-a13*a22)/det
        b32 = (a11*a23-a13*a21)/det
        b33 = (a11*a22-a12*a21)/det

        with open(output_KPOINTS, 'w') as file:
                kpoints_x = ''
                kpoints_y = ''
                kpoints_z = ''
                file.write('Automatic mesh\n')
                file.write( '0\n')
                kpoints_x = round(float(l*Decimal.sqrt(b11*b11+b12*b12+b13*b13)))
                kpoints_y = round(float(l*Decimal.sqrt(b21*b21+b22*b22+b23*b23)))
                kpoints_z = round(float(l*Decimal.sqrt(b31*b31+b32*b32+b33*b33)))
                if TwoDimensional == True:
                        kpoints_z =1
                file.write(MeshType + '\n')
                file.write(str(kpoints_x) + " " + str(kpoints_y) + " " + str(kpoints_z))
        print kpoints_x, kpoints_y, kpoints_z
        
def write_POTCAR():
    os.environ['VASP_PSP_DIR']= '/home/djordje.gluhovic/POTCAR/'
    if os.path.exists('POSCAR'):
        POS = open('POSCAR','r').readlines()
        elements = POS[5].split()
        A = []
        A.append(pseudos.get(elements[0]))
        A.append(pseudos.get(elements[1]))
        POT = Potcar(A)
        POT.write_file('POTCAR')
    else:
        raise IOError('NO POSCAR present')
        
def write_POSCAR_by_mp_id(ID):
    MPR = MPRester('AFLmqRCQTI0hTHNB')
    POS = Poscar(MPR.get_structure_by_material_id(ID))
    POS.write_file('POSCAR')
    
def update_INCAR(names = [], values = []): # Check the significance of vaw line
    if len(names) != len(values):
        raise IOError('Sizes of 2 lists do not match')
        
    if not os.path.exists('INCAR'):
            raise IOError('NO INCAR present')
    INC = Incar.from_file('INCAR')        
    for i in range(len(names)):
        INC.update({names[i]:values[i]})
    
    INC.write_file('INCAR')

def write_relaxation_INCAR():
    INCAR = {'LCHARG': False, 'IBRION': 2, 'PARAM2': 0.22, 'PARAM1': 0.1833333333, 'ISMEAR': 1, 'LWAVE': False, 
    'NPAR': 8, 'SIGMA': 0.1, 'AGGAC': 0.0, 'SYSTEM': 'Cdbr2_3d', 'ENCUT': 500,
    'ISIF': 3, 'GGA': 'Bo', 'EDIFF': 1e-06, 'NSW': 60, 'LAECHG': True, 'LUSE_VDW': True, 'PREC': 'Fast'}
    INC = Incar.from_dict(INCAR)
    INC.write_file('INCAR')
    
    
def check_POTCAR():
    try:
        os.system('grep TITEL POTCAR > A')
        os.system('grep POTCAR: OUTCAR > B')
        
        A = open('A','r').read()
        B = open('B','r').readlines()
        A = re.sub(' +',' ',A)
        for i in range(len(B)):
            B[i] = re.sub('POTCAR: +','',B[i])
            B[i] = re.sub(' +',' ',B[i])
            B[i] = re.sub('\n','',B[i])
            B[i] = B[i].strip()
        if B[0] in A and B[1] in A:
            os.system('rm A B')
            return True
        else:
            os.system('rm A B')
            return False
    except:
        raise IOError('Something is wrong is '+ os.getcwd())
        
def write_KPOINTS(density):
    Str = Structure.from_file('POSCAR')
    KPO = Kpoints.automatic_density(Str, density)
    KPO.write_file('KPOINTS')
 
 
def magnetic_energy(parent_dir):
    #Returns the lower value of the two (magnetic or antifero_magnetic) from parent_directory, and the tag saying which energy is lower, the number of atoms
    magnet = 'magnetism'
    anti = 'anti_fero_magnetism'
    LIST = []
    if not os.path.isdir(parent_dir):
        print 'Wrong directory submited to magnetic_formation_energy()'
        LIST.append(False)
        return LIST
    mag_path = os.path.join(parent_dir,magnet)
    
    if not os.path.isdir(mag_path):
        print 'Parent directory is not a parent directory'
        LIST.append(False)
        return LIST
        
    ant_path = os.path.join(parent_dir,anti)
    if not os.path.isdir(ant_path):
        print 'Parent directory is not a parent directory'
        LIST.append(False)
        return LIST
    
    os.chdir(mag_path)
    if not check_if_done():
        LIST.append(False)
        return LIST
    Energy_mag = float(extract_energy()) 
    
    NUM = num(mag_path)
    
    os.chdir(ant_path)
    if not check_if_done():
        LIST.append(False)
        return LIST
    Energy_ant = float(extract_energy()) 
    
    Min = min(Energy_mag,Energy_ant)
    if Min == Energy_ant:
        tag = 'anti_fero_magnetism'
    elif Min == Energy_mag:
        tag = 'magnetism'
    else:
        raise ValueError('Something is wrong with magnetic_energy()')
    LIST.append(Min)
    LIST.append(tag)
    LIST.append(NUM)
    return LIST
    
def magnetic_formation_energy(twoD_dir,threeD_dir):
    #Returns a string : 'formation energy   2D_tag    3D_tag'. Tags are describing which energy is lower (feromagnetic or antifero_magnetic)
    if not os.path.isdir(twoD_dir) or not os.path.isdir(threeD_dir):
        raise IOError('Directories provided are incorrect')

    
    monolayer_list = magnetic_energy(twoD_dir)
    bulk_list =  magnetic_energy(threeD_dir)
    
    if monolayer_list[0] == False or bulk_list[0] == False:
        return 'Wrong data present'
    
    mono_energy = monolayer_list[0]
    mono_tag = monolayer_list[1]
    mono_num = monolayer_list[2]
    
    bulk_en = bulk_list[0]
    bulk_tag = bulk_list[1]
    bulk_num = bulk_list[2]
    
    formation_energy = (float(mono_energy)/mono_num - float(bulk_en)/bulk_num)*1000
    
    return str(formation_energy)+'  '+mono_tag+ '   '+bulk_tag+'\n'
    
    
def density_of_states_to_excel():
    if not os.path.exists('DOSCAR'):
        print 'DOSCAR not present in '+os.getcwd() 
        return
        
    efermi = Vasprun('vasprun.xml').efermi
    DOS = open('DOSCAR','r').readlines()
    energy, up, down, plot_values = [],[],[],[]
    
    try:
        nedos = Incar.from_file('INCAR').as_dict()['NEDOS'] - 1
    except:
        nedos = 300
    print nedos
        
    try:
        for i in range(7,7+nedos+2):
            A = DOS[i].split()
            energy.append(float(float(A[0])-efermi))
            up.append(float(A[1]))
            down.append(-float(A[2]))

    except:
       print 'Error in indexing in function density_of_states_to_excel()'
    SUM = list(np.array(up)-np.array(down))

    wb = Workbook()
    ws = wb.active  # grab the active worksheet
    ws['A1'] = 'Energy' # Data can be assigned directly to cells
    ws['B1'] = 'Up'
    ws['C1'] = 'Down'
    ws['D1'] = 'Sum'

    for i in range(nedos+1):
        en_str = 'A'+str(i+2)
        up_str = 'B'+str(i+2)
        down_str = 'C'+str(i+2)
        sum_str = 'D'+str(i+2)
        ws[en_str]=float(energy[i])
        ws[up_str]=float(up[i])
        ws[down_str] = float(down[i])
        ws[sum_str] = float(SUM[i])
    chart = ScatterChart()
    chart.title = "Density of states vs normalized Fermi energy\n"+name
    chart.style = 2
    chart.legend.position = 'r'
    xvalues = Reference(ws, min_col=1, min_row=2, max_row=nedos+1)
    for i in range(2, 5):
        values = Reference(ws, min_col=i, min_row=1, max_row=nedos+1)
        series = Series(values, xvalues, title_from_data=True)
        chart.series.append(series)
    plot = wb.create_chartsheet()
    plot.add_chart(chart)

    wb.save('density_of_states.xlsx')
    
def energy_diff_magnetism(DIR):
#This function takes DIR (directory where magnetism and antiferomagnetism direccotories are located) as argument.
#It returns a list that conrains the difference in energies (feromagnetic vs antiferomagnetic) and tag saying which energy is lower 
    A = os.getcwd()
    os.chdir(DIR)
    if not os.path.exists('magnetism') or not os.path.exists('anti_fero_magnetism'):
        return ['False']
    
    magnet_path = os.path.join(DIR,'magnetism')
    anti_path = os.path.join(DIR,'anti_fero_magnetism')
    
    os.chdir(magnet_path)
    if check_if_done():
        mag_energy = extract_energy()
    else:
        mag_energy = 0
        
    
    os.chdir(anti_path)
    if check_if_done():
        anti_energy = extract_energy()
    else:
        anti_energy = 0
    difference = abs(mag_energy-anti_energy)
    if mag_energy <anti_energy:
        tag = 'magnet'
    else:
        tag = 'anti_fero_magnet'
    if mag_energy ==0 or anti_energy ==0:
        tag = False
    os.chdir(A)
    return [difference*1000,tag]
