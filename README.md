#from asyncio.windows_events import NULL
#from pstats import SortKey
#from textwrap import indent
import ensurepip
import simplejson
#from json import JSONEncoder
import json
#from numpy import typename
#from pyparsing import null_debug_action
from openpyxl import load_workbook
from conf.config import GlobalVar as GV
from common.common import get_cell_value
from common.common import re_name


class RowInfo():
    def __init__(self, lev, name, type, maxLen, zenOrHalf, isRequired, title="", description=""):
        self.level = lev
        self.name = name
        self.maxLen = maxLen
        self.zenOrHalf = zenOrHalf
        self.isRequired = isRequired
        self.children = None  # arry or object
        self.type = None
        if type.lower() == 'string':
            self.type = 'string'
            self.data = "default string"
        elif type.lower() == 'number':
            self.data = 1234
            self.type = 'number'
        elif type.lower() == 'array':
            self.data = None
            self.type = 'array'
            self.children = []  # object or single type array
        elif type.lower() == 'date-time':
            self.data = "2022-01-25"
            self.type = 'string'
        else:
            self.data = None
            self.type = 'object'
            self.children = []  # arry or object

        self.requiredList = []
        self.parent = None
        self.childrenType = None
        self.childrenMaxLen = None
        self.title = title
        self.description = description

    # def default(self):
    #    return self.__dict__

    # def toJSON(self):
    def __str__(self):
        myobj = {
            "level": self.level,
            "parent": self.parent.name if self.parent else "NULL",
            "name": self.name,
            "type": self.type,
            "title": self.title,
            "description": self.description
        }
        return json.dumps(myobj, default=lambda o: o.__dict__, sort_keys=True, indent=4)

    def addParent(self, p):
        self.parent = p

    def addChild(self, child):
        if child.name == '-':
            self.childrenType = child.type
            self.childrenMaxLen = child.maxLen
        else:
            self.children.append(child)

    def addRequired(self, requiredName):
        self.requiredList.append(requiredName)

    def getJsonSchema(self):
        ''' create json schema '''
        jsonSchema = {}
        if self.type == "string":
            if self.maxLen != None:
                jsonSchema = {
                    self.name: {
                        #                        "title" : self.title if self.title else "",
                        #                        "description": self.description if self.description else "",
                        "type": "string",
                        "maxLength": self.maxLen
                    }
                }
            else:
                jsonSchema = {
                    self.name: {
                        #                        "title" : self.title if self.title else "",
                        #                        "description": self.description if self.description else "",
                        "type": "string"
                    }
                }
        elif self.type == "number":
            jsonSchema = {
                self.name: {
                    #                        "title" : self.title if self.title else "",
                    #                        "description": self.description if self.description else "",
                    "type": "number"
                }
            }
        elif self.type == "array":
            if self.children and len(self.children) > 0:
                '''array object'''
                prop = {}
                for item in self.children:
                    cj = item.getJsonSchema()
                    for k in cj.keys():
                        prop[k] = cj[k]

                jsonSchema = {
                    self.name: {
                        #                        "title" : self.title if self.title else "",
                        #                        "description": self.description if self.description else "",
                        "type": "array",
                        "items": {
                            "type": "object",
                            "required": [item for item in self.requiredList if item != None],
                            "properties": prop
                        }
                    }
                }
            else:
                ''' simple array '''
                if(self.childrenMaxLen):
                    jsonSchema = {
                        self.name: {
                            #                            "title" : self.title if self.title else "",
                            #                            "description": self.description if self.description else "",
                            "type": "array",
                            "items": {
                                "type": self.childrenType,
                                "maxLength": self.childrenMaxLen
                            }
                        }
                    }
                else:
                    jsonSchema = {
                        self.name: {
                            #                            "title" : self.title if self.title else "",
                            #                            "description": self.description if self.description else "",
                            "type": "array",
                            "items": {
                                "anyOf": [
                                    {
                                        "type": self.childrenType,
                                    }
                                ]
                            }
                        }
                    }
        else:
            ''' object '''
            prop = {}
            for item in self.children:
                cj = item.getJsonSchema()
                for k in cj.keys():
                    prop[k] = cj[k]

            jsonSchema = {
                self.name: {
                    #                    "title" : self.title if self.title else "",
                    #                    "description": self.description if self.description else "",
                    "type": "object",
                    "required": [item for item in self.requiredList if item != None],
                    "properties": prop
                }
            }

        return jsonSchema

# class JsonSchemaObj():
#     def __init__(self, parent, type, requiredList, properties:[], tyepName, maxLength, hanlfOrZen):
#         self.parent = parent
#         self.type = type
#         self.requiredList = requiredList
#         self.properties = properties
#         self.typeName = typename
#         self.maxLength = maxLength


class RowInfoArray():
    def __init__(self):
        self.allRows = []

    def setRow(self, rowInfo: RowInfo):
        self.setParent(rowInfo)
        self.allRows.append(rowInfo)
        # print(rowInfo)

    def setParent(self, curRow: RowInfo):
        for row in reversed(self.allRows):
            if curRow.level > row.level:
                curRow.addParent(row)
                row.addChild(curRow)
                if curRow.isRequired:
                    row.addRequired(curRow.name)
                break

    def getRootJsonObj(self):
        ''' TODO '''
        return

    def getRootJsonSchema(self):
        root = {
            "$schema": "http://json-schema.org/draft-04/schema#",
            "type": "object",
            "properties": self.allRows[0].getJsonSchema()
        }
        return root


class RawRow():
    def __init__(self, oneRowList):
        self.level, self.name, self.arrayPrefix = self.getLevelInfo(
            oneRowList, 10)
        self.itemLogicName = oneRowList[12-1]
        self.isRequired = ("〇" == oneRowList[13-1])
        self.type = oneRowList[14-1]
        self.dataDescription = oneRowList[15-1]
        self.half = oneRowList[16-1]
        self.itemDescription = oneRowList[17-1]
        self.sample = oneRowList[18-1]

    def getLevelInfo(self, oneRowList, toCol):
        level = 0
        name = ""
        arrayPrefix = ""

        for cell in oneRowList:
            if (level >= toCol):
                break
            level += 1
            if cell == '-' and oneRowList[level]:
                name = oneRowList[level]
                arrayPrefix = "-"
                break
            elif cell:
                name = cell
                arrayPrefix = ""
                break
        if arrayPrefix == '-':
            level += 1

        return level, name, arrayPrefix

    def createRowInfo(self):
        maxLen = None
        if self.dataDescription:
            x = self.dataDescription.split(":", 2)
            if x[0] == "maxLength":
                maxLen = int(x[1])

        return RowInfo(self.level, self.name, self.type, maxLen, self.half, self.isRequired, self.itemLogicName, self.itemDescription)


class WsObj():
    def __init__(self, ws, point, maxRow):
        self.ws = ws
        self.start_row_num = point[0]
        self.start_clm_num = point[1]
        self.max_row_num = maxRow

    def get_required(self):
        row_num = self.start_row_num
        clm_num = self.start_clm_num
        required_list = []
        while True:
            row_num += 1
            if get_cell_value(self.ws, row_num, clm_num) is None and get_cell_value(self.ws, row_num, clm_num+1):
                if get_cell_value(self.ws, row_num, GV.REQUIRED_CLM):
                    required_list.append(get_cell_value(
                        self.ws, row_num, clm_num+1))
            if get_cell_value(self.ws, row_num, clm_num) is None and get_cell_value(self.ws, row_num, clm_num+1) is None:
                break
        return required_list

    def readOneRow(self, rowNum):
        rowInfo = []
        for col in range(self.start_clm_num, 20):
            rowInfo.append(self.ws.cell(rowNum, col).value)
        # print(rowInfo)
        return rowInfo

    def readAllRow(self):
        ''' TODO '''
        allRow = RowInfoArray()
        for row in range(self.start_row_num, self.max_row_num):
            rawRow = RawRow(self.readOneRow(row))
            allRow.setRow(rawRow.createRowInfo())

        return allRow.getRootJsonSchema()

    def get_properties(self):
        row_num = self.start_row_num
        clm_num = self.start_clm_num
        schema_name = re_name(get_cell_value(
            self.ws, row_num, clm_num))
        schema_description = get_cell_value(
            self.ws, row_num, GV.DESCRIPTION_CLM)
        required = self.get_required()
        properties = {}
        while True:
            row_num += 1
            if get_cell_value(self.ws, row_num, clm_num) or row_num == GV.MAX_ROW+1:
                break
            if get_cell_value(self.ws, row_num, clm_num+1):
                item_key = get_cell_value(
                    self.ws, row_num, clm_num+1)
                description = get_cell_value(
                    self.ws, row_num, GV.DESCRIPTION_CLM)
                data_type = get_cell_value(self.ws, row_num, GV.TYPE_CLM)
                data_format = get_cell_value(
                    self.ws, row_num, GV.FORMAT_CLM)
                example = get_cell_value(self.ws, row_num, GV.EXAMPLE_CLM)

                properties[item_key] = {
                    "description": description,
                    "type": data_type,
                    "format": data_format,
                    "example": example
                }
                # print(print(chardet.detect(str.encode(description))))

        schema = {
            schema_name: {
                "description": schema_description,
                "type": "object",
                "required": required,
                "properties": properties

            }
        }
        return schema


if __name__ == '__main__':
    wb_path = r'20220126-JsonSchema\CreateJsonFromExcel\excel\API定義書_v1.4.4.0.xlsx'
    ws_name = 'data_registration'

    ws = load_workbook(wb_path)[ws_name]
    start_point = (9, 1)
    obj = WsObj(ws, start_point, 1370)

    # with open('demo.yaml', mode='w', encoding='utf-8') as output_file:
    #     output_file.write(yaml.dump(obj.get_properties(),
    #                                 Dumper=yaml.CDumper, sort_keys=False))
    with open('JsonSchema.json', mode='w', encoding='utf-8') as output_file:
        output_file.write(simplejson.dumps(
            obj.readAllRow(), indent=2, ensure_ascii=False))
