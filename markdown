import * as fs from "fs";
import * as ts from "typescript";
import { NodeInfo } from "./NodeInfo";
export class MarkDownTools {
    public static parameterTableTitles = "parameter | datatype |description";
    public static propertiesTableTitles = "name|datatype|description";
    public static parameterTitles = "#### parameters";
    public static propertyTitles = "### Properties"
    public static tableFormat = "|---|---|---|";
    //转化类方法
    public static transformMethod(node: NodeInfo) {
        if (node.type && node.type != "method") {
            console.log("function classsify wrong");
        } else {
            let returnPackage: string[] = [];
            let methodTitle = "### " + node.name + "()";
            let methodComment = '';
            let methodCharacter = '';
            let returnType = '';
            let parametersMD = '';
            methodComment = ">" + this.getComments(node);
            if (node.returnType) {
                returnType = ">" + "return:" + node.returnType + "\n";
            }
            if (node.childrenDetails) {
                let details = node.childrenDetails;
                for (let i = 0; i < details.length; i++) {
                    if (details[i].isStatic) {
                        if (details[i].isStatic == true) {
                            methodCharacter += ">" + "static" + "\n";
                        }
                    } else if (details[i].currentType == 'syntaxList') {
                        let parameters = details[i].childrenDetails;
                        for (let j = 0; j < parameters.length; j++) {
                            let param = this.transformParameter(parameters[j]);
                            let paramWithDoc = '';
                            if (node.comments) {
                                paramWithDoc = this.commentsMatch(node.comments[0], param);
                            } else if (node.comment) {
                                paramWithDoc = this.commentsMatch(node.comment, param);
                            } else {
                                paramWithDoc = param;
                            }
                            if (j == parameters.length - 1) {
                                parametersMD += paramWithDoc;
                            } else {
                                parametersMD += paramWithDoc + "\n";
                            }

                        }
                    }
                }
            }
            returnPackage.push(methodTitle);
            returnPackage.push(methodComment);
            returnPackage.push(methodCharacter);
            returnPackage.push(returnType);
            if (parametersMD != '') {
                returnPackage.push(this.parameterTitles);
                returnPackage.push(this.parameterTableTitles);
                returnPackage.push(this.tableFormat);
            }
            returnPackage.push(parametersMD);
            return this.package(returnPackage, "\n");
        }

    }

    public static commentsMatch(comment: string, stringMD: string): string {
        let matchResult = '';
        let name = stringMD.split("|")[0];
        let commentStart = comment.search(name);
        if (commentStart >= 0) {
            let commentEnd = comment.indexOf("@", commentStart);
            if (commentEnd > commentStart) {
                matchResult = comment.substring(commentStart + name.length, commentEnd - 1);
            } else {
                matchResult = comment.substring(commentStart + name.length, comment.length - 1);
            }
        } else {
            console.log('not match');
        }
        return stringMD + matchResult + "|";
    }

    public static transformParameter(node: NodeInfo): string {
        let parameterType = '';
        let name = '';
        let parameterPackage: string[] = [];
        if (node.type != 'parameter') {
            console.log('not parameter');
        } else {
            name = node.name;
            if (node.childrenDetails) {
                parameterType = this.transformLamda(node.childrenDetails[0]);
            } else {
                parameterType = node.paramType;
            }
        }
        parameterPackage.push(name);
        parameterPackage.push(parameterType);
        return this.package(parameterPackage, "|");
    }

    public static transformLamda(node: NodeInfo): string {
        let returnType = '';
        let paramString = '';
        if (node.type != 'lambda') {
            console.log('not lambda');
        } else {
            if (node.childrenDetails) {
                let parameters = node.childrenDetails;
                for (let i = 0; i < parameters.length; i++) {
                    let param = this.transformParameter(parameters[i]).split("|", 2);
                    if (i = parameters.length - 1) {
                        paramString += param[0] + ":" + param[1];
                    } else {
                        paramString += param[0] + ":" + param[1] + ",";
                    }
                }

            }
            if (node.returnType) {
                returnType = node.returnType;
            }
        }
        return "(" + paramString + ")" + "=>" + returnType;
    }

    public static getComments(node: NodeInfo): string {
        let comment = '';
        if (node.comments) {
            let tag = -1;
            for (let i = 0; i < node.comments.length; i++) {
                if ((tag = node.comments[i].indexOf("@")) > -1) {
                    comment += node.comments[i].substring(0, tag - 1);
                }
            }
        } else if (node.comment) {
            comment = node.comment;
        }
        return comment;
    }
    //转化类
    public static transformClass(node: NodeInfo): string {
        if (node.type != 'class') {
            console.log('not class');
        } else {
            let returnPackage: string[] = [];
            let className = "## " + node.name;
            let classComment = '';
            let classCharacter = '';
            let classMethods = '';
            let classProperties = '';
            classComment = ">" + this.getComments(node);

            if (node.childrenDetails) {
                let classTree = node.childrenDetails;
                for (let i = 0; i < classTree.length; i++) {
                    let herriage = '';
                    let propertyTemp = '';
                    for (let j = 0; j < classTree[i].childrenDetails.length; j++) {
                        let details = classTree[i].childrenDetails[j]
                        if (details.type == 'method') {
                            classMethods += this.transformMethod(classTree[i].childrenDetails[j]);
                        } else if (details.type == 'implements' || details.type == 'extends') {
                            let derive = details.childrenDetails[0].childrenDetails;
                            if (details.type == 'implements') {
                                herriage += 'implements ';
                            } else {
                                herriage += 'extends ';
                            }
                            for (let k = 0; k < derive.length; k++) {
                                if (k == derive.length - 1) {
                                    herriage += derive[k].name + " ";
                                } else {
                                    herriage += derive[k].name + ", ";
                                }
                            }
                        } else if (details.type == 'property') {
                            let propertyCharacter = '';
                            if (details.childrenDetails) {
                                if (details.childrenDetails[0].isStatic) {
                                    propertyCharacter += "static ";
                                }
                            }
                            if (details.propertyType) {
                                propertyCharacter += details.propertyType;
                            }
                            let propertyPackage: string[] = [];
                            propertyPackage.push(details.name);
                            propertyPackage.push(propertyCharacter);
                            propertyPackage.push(this.getComments(details));
                            if (j == classTree[i].childrenDetails.length - 1) {
                                propertyTemp += this.package(propertyPackage, "|");
                            } else {
                                propertyTemp += this.package(propertyPackage, "|") + "\n";
                            }
                        }
                    }
                    if (herriage != '') {
                        classCharacter = ">" + herriage;
                    }
                    if (propertyTemp != '') {
                        classProperties = this.propertyTitles + "\n" + this.propertiesTableTitles + "\n" + this.tableFormat + "\n" + propertyTemp;
                    }
                }
            }
            returnPackage.push(className);
            returnPackage.push(classComment);
            returnPackage.push(classCharacter);
            returnPackage.push(classMethods);
            returnPackage.push(classProperties);
            return this.package(returnPackage, "\n");
        }
    }
    public static transformSouceFile(sourceFile: NodeInfo): string {
        let nodes = sourceFile.childrenDetails[0].childrenDetails;
        let MDFile = '';
        for (let i = 0; i < nodes.length; i++) {
            switch (nodes[i].type) {
                case 'class':
                    MDFile += this.transformClass(nodes[i]);
                    break;
            }
        }
        return MDFile;
    }

    public static package(info: string[], intervial: string): string {
        let packageRes = '';
        info.forEach((infos) => {
            if (infos && infos != '') {
                packageRes += infos + intervial;
            }

        });
        return packageRes;
    }
}

